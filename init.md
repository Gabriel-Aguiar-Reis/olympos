# Olympos

Plataforma educacional full-stack com fórum e blog de artigos para instrutores e escritores — monolito modular NestJS 11 + Next.js 16.

---

## Stack

| Camada          | Tecnologia                                                                      |
| --------------- | ------------------------------------------------------------------------------- |
| Frontend        | Next.js 16 App Router · React 19 · TanStack Query v5 · shadcn/ui · Tailwind CSS |
| Backend         | Node.js 24 LTS · TypeScript 6 · NestJS 11 · @nestjs/graphql (schema-first)      |
| ORM             | Prisma 7 (schema único)                                                         |
| Auth            | jose (JWT RS256) · next-auth v5                                                 |
| Validação       | Zod 4                                                                           |
| Autorização     | CASL 6 (ABAC)                                                                   |
| Pagamentos      | Abstração via interface (possivelmente Pagar.me)                                |
| Armazenamento   | Abstração via interface (possivelmente Cloudflare R2)                           |
| Certificados    | PDFKit                                                                          |
| Busca (FTS)     | PostgreSQL `tsvector` + índice GIN                                              |
| Testes          | Jest 30 · ts-jest · Supertest                                                   |
| Observabilidade | Pino                                                                            |
| Infra           | Docker Compose                                                                  |
| DevOps          | GitHub Actions (CI/CD)                                                          |

---

## Estrutura do Repositório

```
apps/
  web/              # Next.js 16 App Router (frontend)
  api/              # NestJS 11 monolito modular (backend, porta 4000)
    prisma/
      schema.prisma # Schema único para todo o backend
    src/
      users/        # Módulo: identidade, auth, notificações
      education/    # Módulo: cursos, matrículas, certificados, apostilas
      commerce/     # Módulo: pedidos, pagamentos
      content/      # Módulo: blog, artigos, comentários
      forum/        # Módulo: fórum geral + discussions de curso
      shared/       # Guards, decorators, filtros, Zod schemas, CASL
      app.module.ts
      main.ts
    test/
      unit/
      integration/
packages/
  tsconfig/         # tsconfig.base.json compartilhado
infra/
  docker-compose.yml      # PostgreSQL, Redis (cache opcional)
  docker-compose.test.yml # Container isolado para testes
```

Cada módulo NestJS segue Clean Architecture internamente:

```
src/users/                # exemplo: módulo users
  domain/                 # Entidades, value objects, interfaces (ZERO deps externas)
  application/            # Use cases (services NestJS)
  infrastructure/         # Repositórios Prisma, clientes externos
  interface/              # Resolvers GraphQL, DTOs
  users.module.ts         # NestJS module declaration
```

### Comunicação entre módulos

Sem mensageria. Módulos comunicam-se via **injeção de dependência** do NestJS:

```
Next.js (:3000) → NestJS GraphQL (:4000)
```

Um módulo pode importar o service de outro módulo diretamente (ex: `EducationModule` importa `UsersModule` para buscar dados do instrutor). O Prisma client é compartilhado via um `PrismaModule` global.

---

## Princípios Obrigatórios

- **Domain layer** sem dependências externas (sem Prisma, sem NestJS, sem Node APIs em `domain/`)
- Todo acesso ao banco via interface de repositório definida em `domain/` e implementada em `infrastructure/`
- Schemas GraphQL apenas em arquivos `.graphql` — **nenhuma string inline**
- Sem `any` — jamais
- Autorização via CASL em resolver (Guard NestJS) **e** use case (re-validação com contexto do recurso)
- TDD obrigatório: cobertura Domain ≥ 95%, Application ≥ 85%
- Todos os inputs validados com Zod antes de entrar no use case
- Módulos podem importar services de outros módulos via NestJS DI — sem necessidade de mensageria

---

## Auth & Authorization

### JWT RS256

- **Access token**: 15 minutos, payload `{ sub: userId, roles: Role[] }`
- **Refresh token**: 7 dias, opaco, hash bcrypt armazenado no banco
- Rotação a cada uso (revoga o anterior)
- Auditoria: `ipAddress` + `userAgent` salvo com cada refresh token

### CASL 6 (ABAC)

Duas camadas de verificação:

1. **Guard NestJS** (no resolver): verifica se o role tem _possibilidade_ de executar a ação
2. **Use case**: re-valida com contexto do recurso (ex: "é o instrutor deste curso?")

### Roles

| Role       | Descrição                                                    |
| ---------- | ------------------------------------------------------------ |
| STUDENT    | Default ao registrar; pode comprar cursos, participar        |
| INSTRUCTOR | Cria e gerencia cursos, apostilas                            |
| WRITER     | Cria e publica artigos no blog                               |
| MODERATOR  | Modera fórum (remove, fecha, fixa threads)                   |
| ADMIN      | Acesso total; gerencia roles, bane usuários, publica artigos |

---

## Shared Primitives (src/shared/)

```
I18nString    — JSONB { [locale: BCP47]: string }; pelo menos uma key obrigatória
Money         — { amount: number (centavos, inteiro), currency: ISO4217 }
Slug          — string URL-safe, lowercase, apenas hífens
Email         — RFC-5322, armazenado em lowercase
Password      — min 10 chars, upper + lower + dígito; armazenado como bcrypt (cost 12)
BCP47Locale   — validado contra lista de locales suportados
```

---

## Modelo de Dados

Todos os modelos vivem em um único Prisma schema (`apps/api/prisma/schema.prisma`). FKs reais entre todas as tabelas.

### Users

#### User

| Campo        | Tipo           | Regras                             |
| ------------ | -------------- | ---------------------------------- |
| id           | UUID v4        | imutável, gerado pelo sistema      |
| email        | Email (VO)     | único, lowercase, RFC-5322         |
| passwordHash | string \| null | bcrypt cost 12; null se OAuth-only |
| name         | string         | 1–100 chars                        |
| bio          | string \| null | max 500 chars                      |
| avatarUrl    | URL \| null    | URL validada                       |
| locale       | BCP47          | default `en`                       |
| timezone     | IANA timezone  | ex: `America/Sao_Paulo`            |
| roles        | Role[]         | mínimo STUDENT                     |
| isActive     | boolean        | false = banido / soft-deleted      |
| createdAt    | DateTime       | UTC                                |
| updatedAt    | DateTime       | UTC                                |

**Transição**: `isActive: true → false` (ban) requer role ADMIN.

#### OAuthAccount

| Campo             | Tipo                           | Regras              |
| ----------------- | ------------------------------ | ------------------- |
| id                | UUID v4                        |                     |
| userId            | UUID                           | FK → User           |
| provider          | enum: GOOGLE, GITHUB, FACEBOOK |                     |
| providerAccountId | string                         | unique por provider |
| accessToken       | string (encrypted)             | encriptado at rest  |
| refreshToken      | string \| null (encrypted)     |                     |
| expiresAt         | DateTime \| null               |                     |

#### RefreshToken

| Campo     | Tipo             | Regras                     |
| --------- | ---------------- | -------------------------- |
| id        | UUID v4          |                            |
| userId    | UUID             | FK → User                  |
| tokenHash | string           | bcrypt hash do token opaco |
| expiresAt | DateTime         | 7 dias a partir da emissão |
| revokedAt | DateTime \| null | null = válido              |
| ipAddress | string           | armazenado para auditoria  |
| userAgent | string           | armazenado para auditoria  |

#### Notification

| Campo     | Tipo              | Regras                            |
| --------- | ----------------- | --------------------------------- |
| id        | UUID v4           |                                   |
| userId    | UUID              | FK → User                         |
| type      | enum (ver abaixo) |                                   |
| payload   | JSONB             | type-safe por tipo de notificação |
| readAt    | DateTime \| null  | null = não lida                   |
| createdAt | DateTime          |                                   |

Tipos: `ENROLMENT_CONFIRMED`, `COURSE_COMPLETED`, `ORDER_CONFIRMED`, `THREAD_REPLIED`, `ARTICLE_PUBLISHED`, `CERTIFICATE_READY`

---

### Education

#### Course

| Campo        | Tipo                             | Regras                                       |
| ------------ | -------------------------------- | -------------------------------------------- |
| id           | UUID v4                          |                                              |
| instructorId | UUID                             | FK → User                                    |
| title        | I18nString                       | obrigatório; min 1 key; max 120 chars/locale |
| description  | I18nString                       | obrigatório; max 2000 chars                  |
| thumbnailUrl | URL \| null                      |                                              |
| categoryId   | UUID                             | FK → Category                                |
| status       | enum: DRAFT, PUBLISHED, ARCHIVED | inicia DRAFT                                 |
| price        | Money                            | amount ≥ 0; 0 = gratuito                     |
| language     | BCP47                            | idioma principal do conteúdo                 |
| createdAt    | DateTime                         |                                              |
| updatedAt    | DateTime                         |                                              |

**Transições**:

- `DRAFT → PUBLISHED`: requer ≥ 1 Module com ≥ 1 Lesson
- `PUBLISHED → ARCHIVED`: matriculados mantêm acesso; novas matrículas bloqueadas
- `ARCHIVED → PUBLISHED`: permitido

#### Module

| Campo     | Tipo       | Regras                     |
| --------- | ---------- | -------------------------- |
| id        | UUID v4    |                            |
| courseId  | UUID       | FK → Course                |
| title     | I18nString |                            |
| order     | integer    | ≥ 1; único dentro do curso |
| createdAt | DateTime   |                            |

#### Lesson

| Campo           | Tipo                    | Regras                                             |
| --------------- | ----------------------- | -------------------------------------------------- |
| id              | UUID v4                 |                                                    |
| moduleId        | UUID                    | FK → Module                                        |
| title           | I18nString              |                                                    |
| type            | enum: VIDEO, TEXT, QUIZ |                                                    |
| contentUrl      | URL \| null             | apenas VIDEO                                       |
| contentBody     | JSONB \| null           | TEXT (rich text AST); QUIZ (questões serializadas) |
| durationSeconds | integer \| null         | apenas VIDEO                                       |
| order           | integer                 | ≥ 1; único dentro do módulo                        |
| createdAt       | DateTime                |                                                    |

#### Quiz (embutido em Lesson.contentBody)

```json
{
  "questions": [
    { "prompt": "string", "options": ["string"], "answerIndex": 0 }
  ],
  "passingScore": 70
}
```

Regras: 2–6 opções, `answerIndex` 0-based, `passingScore` 0–100.

#### Category

| Campo            | Tipo         | Regras                  |
| ---------------- | ------------ | ----------------------- |
| id               | UUID v4      |                         |
| name             | I18nString   |                         |
| slug             | Slug         | único                   |
| parentCategoryId | UUID \| null | nullable para top-level |

#### Enrolment

| Campo     | Tipo                                     | Regras          |
| --------- | ---------------------------------------- | --------------- |
| id        | UUID v4                                  |                 |
| courseId  | UUID                                     | FK → Course     |
| userId    | UUID                                     | FK → User       |
| status    | enum: PENDING_PAYMENT, ACTIVE, COMPLETED |                 |
| paymentId | string \| null                           | ID do pagamento |
| createdAt | DateTime                                 |                 |
| updatedAt | DateTime                                 |                 |

**Constraint**: unique(courseId, userId).

**Transições**:

- Gratuito: `→ ACTIVE` imediatamente
- Pago: `→ PENDING_PAYMENT` → `ACTIVE` após pagamento confirmado

#### LessonProgress

| Campo       | Tipo     | Regras         |
| ----------- | -------- | -------------- |
| id          | UUID v4  |                |
| enrolmentId | UUID     | FK → Enrolment |
| lessonId    | UUID     | FK → Lesson    |
| completedAt | DateTime |                |

**Constraint**: unique(enrolmentId, lessonId).
Todas completadas → `Enrolment.status = COMPLETED`.

#### Certificate

| Campo       | Tipo     | Regras                       |
| ----------- | -------- | ---------------------------- |
| id          | UUID v4  |                              |
| enrolmentId | UUID     | FK → Enrolment (unique)      |
| userId      | UUID     | FK → User (desnormalizado)   |
| courseId    | UUID     | FK → Course (desnormalizado) |
| fileUrl     | URL      | URL assinada do storage      |
| issuedAt    | DateTime |                              |

#### CourseAttachment

| Campo         | Tipo           | Regras                                             |
| ------------- | -------------- | -------------------------------------------------- |
| id            | UUID v4        | gerado no request de upload                        |
| courseId      | UUID           | FK → Course                                        |
| displayName   | string         | não-vazio, ≤ 200 chars                             |
| format        | enum: PDF, ODG |                                                    |
| fileSizeBytes | integer        | > 0 e ≤ 50 MB                                      |
| storageKey    | string         | `attachments/{courseId}/{id}.{ext}`; nunca exposto |
| displayOrder  | integer        | ≥ 1, sem gaps                                      |
| uploaderId    | UUID           | FK → User                                          |
| uploadedAt    | DateTime       |                                                    |

---

### Commerce

#### Order

| Campo       | Tipo                                     | Regras         |
| ----------- | ---------------------------------------- | -------------- |
| id          | UUID v4                                  |                |
| userId      | UUID                                     | FK → User      |
| status      | enum: PENDING, PAID, REFUNDED, CANCELLED | inicia PENDING |
| totalAmount | Money                                    | soma dos items |
| createdAt   | DateTime                                 |                |
| updatedAt   | DateTime                                 |                |

**Transições**: PENDING → PAID, PENDING → CANCELLED, PAID → REFUNDED.

#### OrderItem

| Campo       | Tipo         | Regras                                 |
| ----------- | ------------ | -------------------------------------- |
| id          | UUID v4      |                                        |
| orderId     | UUID         | FK → Order                             |
| productType | enum: COURSE | extensível                             |
| productId   | UUID         | referência ao recurso (ex: courseId)   |
| price       | Money        | snapshot do preço no momento da compra |

#### IPaymentProvider (Port — abstração)

```typescript
interface IPaymentProvider {
  createPaymentIntent(order: Order): Promise<PaymentIntent>
  confirmPayment(paymentId: string): Promise<PaymentResult>
  refund(paymentId: string): Promise<RefundResult>
}
```

---

### Content

#### Article

| Campo              | Tipo                           | Regras                   |
| ------------------ | ------------------------------ | ------------------------ |
| id                 | UUID v4                        |                          |
| authorId           | UUID                           | FK → User                |
| title              | I18nString                     |                          |
| slug               | Slug                           | único; auto-gerado       |
| body               | JSONB                          | rich text AST por locale |
| status             | enum: DRAFT, REVIEW, PUBLISHED | inicia DRAFT             |
| readingTimeMinutes | integer                        | computado do word count  |
| commentsEnabled    | boolean                        | default true             |
| language           | BCP47                          |                          |
| publishedAt        | DateTime \| null               | null até PUBLISHED       |
| createdAt          | DateTime                       |                          |
| updatedAt          | DateTime                       |                          |
| searchVector       | tsvector                       | GIN indexed              |

**Transições**: DRAFT → REVIEW → PUBLISHED, REVIEW → DRAFT, PUBLISHED → DRAFT.

#### Comment

| Campo     | Tipo                  | Regras              |
| --------- | --------------------- | ------------------- |
| id        | UUID v4               |                     |
| articleId | UUID                  | FK → Article        |
| authorId  | UUID                  | FK → User           |
| body      | string                | 1–2000 chars; plain |
| status    | enum: ACTIVE, REMOVED |                     |
| createdAt | DateTime              |                     |

#### Tag

| Campo | Tipo    | Regras           |
| ----- | ------- | ---------------- |
| id    | UUID v4 |                  |
| name  | string  | único; lowercase |
| slug  | Slug    | único            |

Relação Article ↔ Tag: N:M via `ArticleTag`.

---

### Forum

#### ForumCategory

| Campo            | Tipo               | Regras                  |
| ---------------- | ------------------ | ----------------------- |
| id               | UUID v4            |                         |
| name             | I18nString         |                         |
| slug             | Slug               | único                   |
| description      | I18nString \| null |                         |
| parentCategoryId | UUID \| null       | nullable para top-level |
| order            | integer            | ordem de exibição       |

#### Thread

| Campo      | Tipo                        | Regras             |
| ---------- | --------------------------- | ------------------ |
| id         | UUID v4                     |                    |
| categoryId | UUID                        | FK → ForumCategory |
| authorId   | UUID                        | FK → User          |
| title      | string                      | 5–200 chars        |
| body       | JSONB                       | rich text AST      |
| status     | enum: OPEN, CLOSED, REMOVED | OPEN por default   |
| isPinned   | boolean                     | MODERATOR / ADMIN  |
| voteCount  | integer                     | desnormalizado     |
| replyCount | integer                     | desnormalizado     |
| createdAt  | DateTime                    |                    |
| updatedAt  | DateTime                    |                    |

**Transições**: OPEN → CLOSED (MOD/ADMIN), OPEN/CLOSED → REMOVED (MOD/ADMIN, motivo obrigatório).

#### Reply

| Campo     | Tipo                  | Regras             |
| --------- | --------------------- | ------------------ |
| id        | UUID v4               |                    |
| threadId  | UUID                  | FK → Thread        |
| authorId  | UUID                  | FK → User          |
| body      | JSONB                 | rich text AST      |
| status    | enum: ACTIVE, REMOVED | ACTIVE por default |
| voteCount | integer               | desnormalizado     |
| createdAt | DateTime              |                    |
| updatedAt | DateTime              |                    |

#### Vote

| Campo      | Tipo                | Regras         |
| ---------- | ------------------- | -------------- |
| id         | UUID v4             |                |
| userId     | UUID                | FK → User      |
| targetType | enum: THREAD, REPLY |                |
| targetId   | UUID                | FK polimórfico |
| createdAt  | DateTime            |                |

**Constraint**: unique(userId, targetType, targetId).

#### ModerationLog

| Campo       | Tipo                     | Regras                    |
| ----------- | ------------------------ | ------------------------- |
| id          | UUID v4                  |                           |
| moderatorId | UUID                     | FK → User                 |
| targetType  | enum: THREAD, REPLY      |                           |
| targetId    | UUID                     |                           |
| action      | enum: REMOVE, CLOSE, PIN |                           |
| reason      | string                   | obrigatório; min 10 chars |
| createdAt   | DateTime                 |                           |

#### Discussion (vinculada a curso)

| Campo      | Tipo               | Regras                   |
| ---------- | ------------------ | ------------------------ |
| id         | UUID v4            |                          |
| courseId   | UUID               | FK → Course              |
| authorId   | UUID               | FK → User                |
| title      | string             | 1–300 chars              |
| body       | string             | markdown, 1–50.000 chars |
| status     | enum: OPEN, CLOSED | OPEN por default         |
| isPinned   | boolean            | default false            |
| replyCount | integer            | desnormalizado           |
| createdAt  | DateTime           |                          |
| updatedAt  | DateTime           |                          |

**Invariantes**: CLOSED bloqueia respostas. Apenas instrutor do curso pode close/pin (CASL).

#### DiscussionReply

| Campo        | Tipo     | Regras                   |
| ------------ | -------- | ------------------------ |
| id           | UUID v4  |                          |
| discussionId | UUID     | FK → Discussion          |
| authorId     | UUID     | FK → User                |
| body         | string   | markdown, 1–50.000 chars |
| createdAt    | DateTime |                          |
| updatedAt    | DateTime |                          |

#### CourseMembershipProjection (read model)

Tabela local que projeta a membership de cada curso. Atualizada sincronamente nos use cases de matrícula e publicação de curso — sem necessidade de mensageria.

| Campo    | Tipo                   | Regras                                              |
| -------- | ---------------------- | --------------------------------------------------- |
| courseId | UUID                   | FK → Course                                         |
| userId   | UUID                   | FK → User                                           |
| role     | enum: STUDENT, TEACHER |                                                     |
| isActive | boolean                | false quando matrícula cancelada ou curso arquivado |

**Constraint**: unique(courseId, userId).

**Atualização**:

- `enrolInCourse` (ACTIVE) → upsert STUDENT, isActive = true
- `cancelEnrolment` → isActive = false
- `publishCourse` → upsert TEACHER (instrutor), isActive = true
- `archiveCourse` → isActive = false para o TEACHER

---

## GraphQL API

Schema único exposto pelo NestJS em `:4000/graphql`. Sem federation — schema-first com `@nestjs/graphql`.

### Queries

```graphql
# Users
me: User
user(id: ID!): User
notifications(first: Int, after: String): NotificationConnection

# Education
courses(filter: CourseFilter, first: Int, after: String): CourseConnection
course(id: ID!): Course
searchCourses(query: String!, first: Int, after: String): CourseConnection
categories: [Category!]!
myEnrolments(first: Int, after: String): EnrolmentConnection
enrolment(courseId: ID!): Enrolment
lessonProgress(enrolmentId: ID!): [LessonProgress!]!
certificate(enrolmentId: ID!): Certificate
courseAttachments(courseId: ID!): [CourseAttachment!]!

# Commerce
order(id: ID!): Order
myOrders(first: Int, after: String): OrderConnection

# Content
articles(filter: ArticleFilter, first: Int, after: String): ArticleConnection
article(slug: String!): Article
searchArticles(query: String!, first: Int, after: String): ArticleConnection
tags: [Tag!]!
comments(articleId: ID!, first: Int, after: String): CommentConnection

# Forum
forumCategories: [ForumCategory!]!
threads(categoryId: ID!, first: Int, after: String): ThreadConnection
thread(id: ID!): Thread
replies(threadId: ID!, first: Int, after: String): ReplyConnection
courseDiscussions(courseId: ID!, filter: DiscussionFilter, search: String, first: Int, after: String): DiscussionConnection
discussion(id: ID!): Discussion
discussionReplies(discussionId: ID!, first: Int, after: String): DiscussionReplyConnection
```

### Mutations

```graphql
# Auth
register(input: RegisterInput!): AuthPayload
login(input: LoginInput!): AuthPayload
refreshToken(token: String!): AuthPayload
logout: Boolean

# Users
updateProfile(input: UpdateProfileInput!): User
changePassword(input: ChangePasswordInput!): Boolean
assignRole(userId: ID!, role: Role!): User                    # ADMIN
banUser(userId: ID!): User                                     # ADMIN
markNotificationRead(id: ID!): Notification
markAllNotificationsRead: Boolean

# Education — Courses
createCourse(input: CreateCourseInput!): Course                # INSTRUCTOR
updateCourse(id: ID!, input: UpdateCourseInput!): Course       # INSTRUCTOR owner
publishCourse(id: ID!): Course                                 # INSTRUCTOR owner
archiveCourse(id: ID!): Course                                 # INSTRUCTOR owner
createModule(courseId: ID!, input: CreateModuleInput!): Module
updateModule(id: ID!, input: UpdateModuleInput!): Module
deleteModule(id: ID!): Boolean
reorderModules(courseId: ID!, moduleIds: [ID!]!): [Module!]!
createLesson(moduleId: ID!, input: CreateLessonInput!): Lesson
updateLesson(id: ID!, input: UpdateLessonInput!): Lesson
deleteLesson(id: ID!): Boolean
reorderLessons(moduleId: ID!, lessonIds: [ID!]!): [Lesson!]!

# Education — Enrolment & Progress
enrolInCourse(courseId: ID!): Enrolment
completeLesson(enrolmentId: ID!, lessonId: ID!): LessonProgress

# Education — Attachments
requestAttachmentUploadUrl(input: RequestUploadInput!): UploadUrlPayload
confirmAttachmentUpload(id: ID!): CourseAttachment
renameAttachment(id: ID!, displayName: String!): CourseAttachment
reorderAttachments(courseId: ID!, attachmentIds: [ID!]!): [CourseAttachment!]!
deleteAttachment(id: ID!): Boolean

# Commerce
createOrder(input: CreateOrderInput!): Order
cancelOrder(id: ID!): Order
requestRefund(id: ID!): Order                                  # ADMIN

# Content
createArticle(input: CreateArticleInput!): Article             # WRITER
updateArticle(id: ID!, input: UpdateArticleInput!): Article    # WRITER owner
submitForReview(id: ID!): Article                              # WRITER owner
publishArticle(id: ID!): Article                               # ADMIN
rejectArticle(id: ID!): Article                                # ADMIN
unpublishArticle(id: ID!): Article                             # ADMIN
createComment(articleId: ID!, body: String!): Comment
removeComment(id: ID!): Comment                                # MODERATOR
createTag(input: CreateTagInput!): Tag                         # ADMIN
addTagToArticle(articleId: ID!, tagId: ID!): Article
removeTagFromArticle(articleId: ID!, tagId: ID!): Article

# Forum
createThread(input: CreateThreadInput!): Thread
closeThread(id: ID!, reason: String!): Thread                  # MODERATOR/ADMIN
pinThread(id: ID!): Thread                                     # MODERATOR/ADMIN
unpinThread(id: ID!): Thread                                   # MODERATOR/ADMIN
removeThread(id: ID!, reason: String!): Thread                 # MODERATOR/ADMIN
createReply(threadId: ID!, input: CreateReplyInput!): Reply
removeReply(id: ID!, reason: String!): Reply                   # MODERATOR/ADMIN
vote(targetType: VoteTarget!, targetId: ID!): Vote
unvote(targetType: VoteTarget!, targetId: ID!): Boolean

# Discussions (curso)
createDiscussion(input: CreateDiscussionInput!): Discussion    # course member
closeDiscussion(id: ID!): Discussion                           # INSTRUCTOR do curso
pinDiscussion(id: ID!): Discussion                             # INSTRUCTOR do curso
unpinDiscussion(id: ID!): Discussion                           # INSTRUCTOR do curso
createDiscussionReply(discussionId: ID!, body: String!): DiscussionReply  # course member
```

---

## Relationship Map

```
User  ──(instructorId)──▶  Course
      ──(userId)────────▶  Enrolment
      ──(userId)────────▶  Certificate
      ──(uploaderId)────▶  CourseAttachment
      ──(userId)────────▶  Order
      ──(authorId)──────▶  Article
      ──(authorId)──────▶  Comment
      ──(authorId)──────▶  Thread
      ──(authorId)──────▶  Reply
      ──(userId)────────▶  Vote
      ──(authorId)──────▶  Discussion
      ──(moderatorId)───▶  ModerationLog

Course ──(courseId)──▶  Discussion
       ──(courseId)──▶  CourseMembershipProjection
```

---

## Observabilidade

| Pilar   | Ferramenta | Detalhes                     |
| ------- | ---------- | ---------------------------- |
| Logging | Pino       | JSON estruturado, request-id |

O backend expõe `/health` (liveness check).

---

## Convenções de Commit

[Conventional Commits](https://www.conventionalcommits.org/) com Commitlint + Husky:

```
feat(education): add certificate generation
fix(users): prevent duplicate refresh token rotation
test(forum): add vote uniqueness invariant tests
chore: bump prisma to 7.x
```

---

## Licença

Privado — todos os direitos reservados.
