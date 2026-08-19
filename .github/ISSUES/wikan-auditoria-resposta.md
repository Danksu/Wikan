# 📋 Resposta à Issue: Plano de Mitigação e Implementação

Obrigado pela auditoria extremamente detalhada e crítica. Esta resposta apresenta o plano de ação para mitigar cada problema identificado, organizado por prioridade e fase de implementação.

---

## 🎯 Fase 1: Fundação Segura (Prioridade Crítica)

### 1.1 Stack Tecnológica Definida

**Decisão Arquitetural**:
```
Backend:   Node.js + NestJS (TypeScript)
Frontend:  Next.js 14+ (React, App Router, SSR)
Banco:     PostgreSQL 15+
ORM:       Prisma
Cache:     Redis
Auth:      NextAuth.js com provedor Google
Storage:   AWS S3 ou compatível (MinIO self-hosted)
Busca:     PostgreSQL Full-Text (inicial) → Meilisearch (escala)
Deploy:    Docker + Docker Compose (dev), Kubernetes (prod futuro)
CI/CD:     GitHub Actions
```

**Justificativa**:
- TypeScript oferece tipagem estática, reduzindo erros
- NestJS fornece arquitetura modular e injeção de dependência
- Next.js garante SSR para SEO
- Prisma facilita migrations e type-safety
- PostgreSQL é robusto e possui full-text search nativo

### 1.2 Sistema de Permissões RBAC

**Modelo Proposto**:

```prisma
model Role {
  id          String       @id @default(uuid())
  name        String       @unique // 'visitor', 'user', 'editor', 'admin'
  description String?
  permissions Permission[]
  users       User[]
  createdAt   DateTime     @default(now())
  updatedAt   DateTime     @updatedAt
}

model Permission {
  id          String @id @default(uuid())
  name        String @unique // 'page.create', 'page.edit', 'page.delete', 'user.manage'
  description String?
  roles       Role[]
  resource    String // 'pages', 'users', 'settings'
  action      String // 'create', 'read', 'update', 'delete'
}

model User {
  id            String    @id @default(uuid())
  email         String    @unique
  googleId      String    @unique
  name          String?
  avatar        String?
  roles         Role[]    @relation("UserRoles")
  editorRequest EditorRequest?
  // ... outros campos
}
```

**Implementação**:
```typescript
// guards/permissions.guard.ts
@Injectable()
export class PermissionsGuard implements CanActivate {
  constructor(
    private reflector: Reflector,
    private permissionService: PermissionService
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const requiredPermissions = this.reflector.getAllAndOverride<string[]>('permissions', [
      context.getHandler(),
      context.getClass(),
    ]);

    if (!requiredPermissions) return true;

    const request = context.switchToHttp().getRequest();
    const user = request.user;

    return this.permissionService.hasPermissions(user.id, requiredPermissions);
  }
}

// Uso nos controllers
@Controller('pages')
@UseGuards(PermissionsGuard)
export class PagesController {
  @Post()
  @Permissions('page.create')
  createPage(@Body() dto: CreatePageDto) {
    // ...
  }
}
```

### 1.3 Segurança - XSS Prevention

**Estratégia Multi-Camada**:

```typescript
// 1. Sanitização no backend
import DOMPurify from 'dompurify';
import { JSDOM } from 'jsdom';

const window = new JSDOM('').window;
const purify = DOMPurify(window);

@Injectable()
export class ContentSanitizer {
  private allowedTags = [
    'h1', 'h2', 'h3', 'h4', 'h5', 'h6',
    'p', 'br', 'hr',
    'strong', 'em', 'u', 's',
    'ul', 'ol', 'li',
    'a', 'img',
    'blockquote', 'code', 'pre',
    'table', 'thead', 'tbody', 'tr', 'th', 'td'
  ];

  sanitize(content: string): string {
    return purify.sanitize(content, {
      ALLOWED_TAGS: this.allowedTags,
      ALLOWED_ATTR: ['href', 'src', 'alt', 'title', 'target', 'rel'],
      ADD_ATTR: ['target'],
      FORBID_ATTR: ['onclick', 'onerror', 'onload', 'onmouseover']
    });
  }
}

// 2. CSP Headers
app.use((req, res, next) => {
  res.setHeader(
    'Content-Security-Policy',
    "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self' data:;"
  );
  next();
});

// 3. Frontend - nunca usar dangerouslySetInnerHTML sem sanitização
function SafeHTML({ content }: { content: string }) {
  const sanitized = useMemo(() => DOMPurify.sanitize(content), [content]);
  return <div dangerouslySetInnerHTML={{ __html: sanitized }} />;
}
```

### 1.4 Upload Seguro de Imagens

```typescript
// services/upload.service.ts
@Injectable()
export class UploadService {
  private readonly ALLOWED_MIME_TYPES = [
    'image/jpeg',
    'image/png',
    'image/webp',
    'image/gif'
  ];

  private readonly MAX_FILE_SIZE = 5 * 1024 * 1024; // 5MB

  async validateAndUpload(file: Express.Multer.File, userId: string) {
    // 1. Validar MIME type real (não confiar na extensão)
    const fileType = await FileType.fromBuffer(file.buffer);
    
    if (!fileType || !this.ALLOWED_MIME_TYPES.includes(fileType.mime)) {
      throw new BadRequestException('Tipo de arquivo não permitido');
    }

    // 2. Validar tamanho
    if (file.buffer.length > this.MAX_FILE_SIZE) {
      throw new BadRequestException('Arquivo muito grande (máx 5MB)');
    }

    // 3. Re-processar imagem (remove metadata e possíveis scripts)
    const processedImage = await this.reprocessImage(file.buffer);

    // 4. Gerar nome seguro (UUID, nunca usar nome original)
    const filename = `${uuid()}.${this.getExtension(fileType.mime)}`;

    // 5. Armazenar fora do webroot (S3 bucket privado)
    const url = await this.uploadToS3(processedImage, filename);

    // 6. Gerar thumbnail
    const thumbnail = await this.generateThumbnail(processedImage);
    await this.uploadToS3(thumbnail, `thumb_${filename}`);

    return { url, thumbnail };
  }

  private async reprocessImage(buffer: Buffer): Promise<Buffer> {
    return sharp(buffer)
      .rotate() // Remove orientação EXIF
      .resize({ width: 2000, height: 2000, fit: 'inside' }) // Limita dimensões
      .toFormat('webp', { quality: 85 }) // Converte para WebP
      .toBuffer();
  }
}
```

### 1.5 Tratamento de Edição Simultânea

```typescript
// services/page.service.ts
@Injectable()
export class PageService {
  async updatePage(
    pageId: string,
    dto: UpdatePageDto,
    userId: string,
    expectedRevisionId: string // Optimistic locking
  ) {
    return this.prisma.$transaction(async (tx) => {
      // 1. Verificar revisão atual
      const currentPage = await tx.page.findUnique({
        where: { id: pageId },
        include: { currentRevision: true }
      });

      if (!currentPage) {
        throw new NotFoundException('Página não encontrada');
      }

      // 2. Detectar conflito
      if (currentPage.currentRevisionId !== expectedRevisionId) {
        throw new ConflictException(
          'Esta página foi modificada por outro usuário. Por favor, recarregue e tente novamente.',
          {
            currentRevision: currentPage.currentRevision,
            yourRevision: expectedRevisionId
          }
        );
      }

      // 3. Criar nova revisão
      const newRevision = await tx.pageRevision.create({
        data: {
          content: dto.content,
          summary: dto.summary,
          authorId: userId,
          pageId: pageId,
          version: currentPage.currentRevision.version + 1
        }
      });

      // 4. Atualizar página
      const updatedPage = await tx.page.update({
        where: { id: pageId },
        data: {
          currentRevisionId: newRevision.id,
          updatedAt: new Date(),
          updatedById: userId
        }
      });

      // 5. Invalidar cache
      await this.cacheService.del(`page:${pageId}`);

      return updatedPage;
    });
  }
}
```

### 1.6 Rate Limiting

```typescript
// middleware/rate-limit.middleware.ts
@Injectable()
export class RateLimitMiddleware implements NestMiddleware {
  private loginLimiter = rateLimit({
    windowMs: 60 * 60 * 1000, // 1 hora
    max: 5, // 5 tentativas
    message: 'Muitas tentativas de login. Tente novamente em 1 hora.',
    keyGenerator: (req) => req.ip || 'unknown'
  });

  private editorRequestLimiter = rateLimit({
    windowMs: 24 * 60 * 60 * 1000, // 24 horas
    max: 3, // 3 solicitações por dia
    message: 'Muitas solicitações de editor. Tente amanhã.',
    keyGenerator: (req) => req.user?.id || req.ip
  });

  private apiLimiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutos
    max: 100, // 100 requisições
    message: 'Muitas requisições. Diminua a frequência.'
  });

  use(req: Request, res: Response, next: NextFunction) {
    if (req.path === '/api/auth/login') {
      return this.loginLimiter(req, res, next);
    }
    
    if (req.path === '/api/editor-requests') {
      return this.editorRequestLimiter(req, res, next);
    }

    return this.apiLimiter(req, res, next);
  }
}
```

### 1.7 Backups Automatizados

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:15
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./backups:/backups
  
  backup:
    image: postgres:15
    environment:
      - PGPASSWORD=${DATABASE_PASSWORD}
    command: >
      bash -c "
      while true; do
        pg_dump -h postgres -U ${DATABASE_USER} ${DATABASE_NAME} | gzip > /backups/wikan-backup-$$(date +\%Y\%m\%d-\%H\%M).sql.gz
        find /backups -name '*.sql.gz' -mtime +7 -delete
        sleep 86400
      done"
    volumes:
      - ./backups:/backups
```

```bash
# scripts/restore-backup.sh
#!/bin/bash
if [ -z "$1" ]; then
  echo "Uso: ./restore-backup.sh <backup-file.sql.gz>"
  exit 1
fi

gunzip -c "$1" | psql -h localhost -U postgres wikan
echo "Backup restaurado com sucesso!"
```

---

## 🏗️ Fase 2: Modelagem de Dados Completa

### Schema Prisma Completo

```prisma
// schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

enum UserRole {
  VISITOR
  USER
  EDITOR
  ADMIN
}

enum RequestStatus {
  PENDING
  APPROVED
  REJECTED
}

enum PageStatus {
  PUBLISHED
  DRAFT
  DELETED
}

model User {
  id            String    @id @default(uuid())
  email         String    @unique
  googleId      String    @unique
  name          String?
  avatar        String?
  bio           String?
  role          UserRole  @default(USER)
  isBlocked     Boolean   @default(false)
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
  lastLoginAt   DateTime?
  
  editorRequest EditorRequest?
  createdPages  Page[]    @relation("PageAuthor")
  updatedPages  Page[]    @relation("PageUpdater")
  revisions     PageRevision[]
  follows       Follow[]
  notifications Notification[]
  auditLogs     AuditLog[]
  
  @@index([email])
  @@index([googleId])
  @@index([role])
}

model EditorRequest {
  id          String       @id @default(uuid())
  userId      String       @unique
  user        User         @relation(fields: [userId], references: [id])
  motivation  String
  status      RequestStatus @default(PENDING)
  reviewedBy  String?
  reviewedAt  DateTime?
  reason      String?      // Para rejeição
  createdAt   DateTime     @default(now())
  updatedAt   DateTime     @updatedAt

  @@index([status])
  @@index([createdAt])
}

model Page {
  id              String   @id @default(uuid())
  wikiId          String?  // Para multi-tenancy futuro
  slug            String
  title           String
  currentRevisionId String?
  status          PageStatus @default(PUBLISHED)
  createdById     String
  createdBy       User     @relation("PageAuthor", fields: [createdById], references: [id])
  updatedById     String?
  updatedBy       User?    @relation("PageUpdater", fields: [updatedById], references: [id])
  categoryId      String?
  category        Category? @relation(fields: [categoryId], references: [id])
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt
  deletedAt       DateTime?
  
  revisions       PageRevision[]
  categories      Category[] @relation("PageCategories")
  follows         Follow[]
  
  @@unique([wikiId, slug])
  @@index([slug])
  @@index([status])
  @@index([categoryId])
  @@index([createdAt])
  @@index([deletedAt])
}

model PageRevision {
  id        String   @id @default(uuid())
  pageId    String
  page      Page     @relation(fields: [pageId], references: [id])
  version   Int
  content   String   @db.Text
  summary   String?
  authorId  String
  author    User     @relation(fields: [authorId], references: [id])
  createdAt DateTime @default(now())
  
  @@unique([pageId, version])
  @@index([pageId, createdAt])
  @@index([authorId])
}

model Category {
  id          String   @id @default(uuid())
  slug        String   @unique
  name        String
  description String?
  parentId    String?
  parent      Category? @relation("CategoryHierarchy", fields: [parentId], references: [id])
  children    Category[] @relation("CategoryHierarchy")
  pages       Page[]   @relation("PageCategories")
  createdAt   DateTime @default(now())
  
  @@index([parentId])
  @@index([slug])
}

model Follow {
  id        String   @id @default(uuid())
  userId    String
  user      User     @relation(fields: [userId], references: [id])
  pageId    String
  page      Page     @relation(fields: [pageId], references: [id])
  createdAt DateTime @default(now())
  
  @@unique([userId, pageId])
  @@index([userId])
  @@index([pageId])
}

model Notification {
  id        String   @id @default(uuid())
  userId    String
  user      User     @relation(fields: [userId], references: [id])
  type      String   // 'EDITOR_APPROVED', 'PAGE_EDITED', etc.
  title     String
  message   String
  read      Boolean  @default(false)
  link      String?
  createdAt DateTime @default(now())
  
  @@index([userId, read])
  @@index([createdAt])
}

model AuditLog {
  id        String   @id @default(uuid())
  actorId   String?
  actor     User?    @relation(fields: [actorId], references: [id])
  action    String   // 'USER_BLOCKED', 'PAGE_DELETED', etc.
  targetType String  // 'USER', 'PAGE', 'CATEGORY'
  targetId  String
  metadata  Json?
  ipAddress String?
  createdAt DateTime @default(now())
  
  @@index([actorId])
  @@index([targetType, targetId])
  @@index([createdAt])
  @@index([action])
}

model Session {
  id           String   @id @default(uuid())
  userId       String
  user         User     @relation(fields: [userId], references: [id])
  token        String   @unique
  expiresAt    DateTime
  createdAt    DateTime @default(now())
  
  @@index([userId])
  @@index([token])
}
```

---

## 🧪 Fase 3: Estratégia de Testes

### Matriz de Testes de Permissão

```typescript
// tests/permissions/permissions.matrix.test.ts
describe('Permission Matrix', () => {
  const testCases = [
    // { role, endpoint, method, expectedStatus }
    { role: 'visitor', endpoint: '/api/pages', method: 'GET', expected: 200 },
    { role: 'visitor', endpoint: '/api/pages', method: 'POST', expected: 403 },
    { role: 'visitor', endpoint: '/api/pages/1', method: 'PUT', expected: 403 },
    { role: 'visitor', endpoint: '/api/editor-requests', method: 'POST', expected: 401 },
    
    { role: 'user', endpoint: '/api/pages', method: 'GET', expected: 200 },
    { role: 'user', endpoint: '/api/pages', method: 'POST', expected: 403 },
    { role: 'user', endpoint: '/api/editor-requests', method: 'POST', expected: 201 },
    
    { role: 'editor', endpoint: '/api/pages', method: 'POST', expected: 201 },
    { role: 'editor', endpoint: '/api/pages/1', method: 'PUT', expected: 200 },
    { role: 'editor', endpoint: '/api/admin/users', method: 'GET', expected: 403 },
    
    { role: 'admin', endpoint: '/api/admin/users', method: 'GET', expected: 200 },
    { role: 'admin', endpoint: '/api/admin/editor-requests/1/approve', method: 'POST', expected: 200 },
  ];

  testCases.forEach(({ role, endpoint, method, expected }) => {
    it(`${method} ${endpoint} as ${role} should return ${expected}`, async () => {
      const user = await createUserWithRole(role);
      const response = await request(app)[method.toLowerCase()](endpoint)
        .set('Authorization', `Bearer ${user.token}`);
      
      expect(response.status).toBe(expected);
    });
  });
});
```

### Testes de Segurança

```typescript
// tests/security/xss.test.ts
describe('XSS Prevention', () => {
  it('should sanitize malicious scripts', async () => {
    const maliciousContent = `
      <p>Normal text</p>
      <script>alert('XSS')</script>
      <img src="x" onerror="alert('XSS')">
      <a href="javascript:alert('XSS')">Click me</a>
    `;

    const response = await request(app)
      .post('/api/pages')
      .set('Authorization', `Bearer ${editorToken}`)
      .send({
        title: 'Test Page',
        slug: 'test-page',
        content: maliciousContent
      });

    const page = await prisma.page.findUnique({
      where: { id: response.body.id },
      include: { currentRevision: true }
    });

    expect(page.currentRevision.content).not.toContain('<script>');
    expect(page.currentRevision.content).not.toContain('onerror=');
    expect(page.currentRevision.content).not.toContain('javascript:');
    expect(page.currentRevision.content).toContain('<p>Normal text</p>');
  });
});

// tests/security/race-condition.test.ts
describe('Race Condition Prevention', () => {
  it('should prevent concurrent edits from overwriting', async () => {
    const page = await createTestPage();
    const revisionId = page.currentRevisionId;

    // Simular duas edições simultâneas
    const [result1, result2] = await Promise.allSettled([
      updatePage(page.id, { content: 'Edit 1' }, revisionId),
      updatePage(page.id, { content: 'Edit 2' }, revisionId)
    ]);

    expect(result1.status).toBe('fulfilled');
    expect(result2.status).toBe('rejected');
    expect(result2.reason.name).toBe('ConflictException');
  });
});
```

---

## 📊 Fase 4: Monitoramento e Observabilidade

```typescript
// middleware/logging.middleware.ts
@Injectable()
export class LoggingMiddleware implements NestMiddleware {
  private readonly logger = new Logger('HTTP');

  use(req: Request, res: Response, next: NextFunction) {
    const start = Date.now();

    res.on('finish', () => {
      const duration = Date.now() - start;
      
      this.logger.log(
        `${req.method} ${req.originalUrl} ${res.statusCode} - ${duration}ms`,
        {
          ip: req.ip,
          userId: req.user?.id,
          userAgent: req.headers['user-agent']
        }
      );

      // Enviar métricas para Prometheus/New Relic
      metrics.httpRequestDuration.observe(duration / 1000);
      metrics.httpRequestsTotal.inc({
        method: req.method,
        path: req.route?.path || req.path,
        status: res.statusCode
      });
    });

    next();
  }
}
```

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'wikan-backend'
    static_configs:
      - targets: ['backend:3000']
    metrics_path: '/metrics'
```

---

## ✅ Checklist de Validação Final

### Segurança
- [ ] Teste de penetração básico realizado
- [ ] Todos os endpoints protegidos com autorização
- [ ] XSS testado com payloads reais
- [ ] CSRF tokens implementados
- [ ] Rate limiting ativo em endpoints críticos
- [ ] Uploads validados e re-processados
- [ ] Secrets em variáveis de ambiente
- [ ] Logs não contêm dados sensíveis

### Performance
- [ ] Índices criados em todas as chaves de busca
- [ ] N+1 queries eliminadas (usar include/join)
- [ ] Cache implementado para páginas públicas
- [ ] Imagens otimizadas (WebP, thumbnails)
- [ ] Paginação em todas as listagens

### UX
- [ ] Estados de loading visíveis
- [ ] Mensagens de erro amigáveis
- [ ] Empty states implementados
- [ ] Responsividade testada em mobile
- [ ] Contraste de cores ≥ 4.5:1
- [ ] Navegação por teclado funcional

### DevOps
- [ ] CI/CD pipeline configurado
- [ ] Backups automatizados testados
- [ ] Rollback procedure documentada
- [ ] Monitoring e alertas configurados
- [ ] Documentação de deploy completa

---

## 📅 Timeline Estimada

| Fase | Duração | Entregáveis |
|------|---------|-------------|
| Fase 1 | 2 semanas | Auth, RBAC, Segurança básica, Upload |
| Fase 2 | 1 semana | Schema completo, Migrations |
| Fase 3 | 2 semanas | CRUD Páginas, Revisões, Editor |
| Fase 4 | 1 semana | Editor Requests, Aprovações |
| Fase 5 | 1 semana | Dashboard Admin, Moderação |
| Fase 6 | 1 semana | Testes, QA, Security audit |
| Fase 7 | 1 semana | Docs, Deploy, Monitoring |

**Total**: 9 semanas para MVP funcional e seguro

---

## 🔔 Próximos Passos Imediatos

1. **Configurar repositório** com estrutura de pastas modular
2. **Inicializar projeto NestJS + Next.js**
3. **Configurar banco PostgreSQL** local e Docker
4. **Implementar autenticação Google** via NextAuth
5. **Criar migrations iniciais** do schema
6. **Implementar middleware de permissões**
7. **Configurar CI/CD** básico no GitHub Actions

---

**Responsável**: Equipe de Desenvolvimento WIKAN  
**Data de Início**: Imediata  
**Revisão**: Semanal (toda sexta-feira)  
**Próxima Milestone**: Autenticação funcional + Schema DB até [data + 2 semanas]
