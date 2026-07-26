# Discord Insider — техническая спецификация проекта

> Этот документ описывает архитектуру полноценной production-платформы согласно ТЗ. Дизайн-прототип (`index.html`, `article.html`) уже готов и показывает финальный визуальный язык. Ниже — план, по которому можно поднять реальный fullstack-проект (например, через Claude Code, в среде с доступом к сети, PostgreSQL, Redis и S3).

---

## 1. Стек и структура репозитория (Feature-Sliced Design)

```
discord-insider/
├── apps/
│   ├── web/                 # Next.js 15 + React 19 + TS
│   │   └── src/
│   │       ├── app/          # роуты (news, rumors, article/[slug], admin, profile)
│   │       ├── entities/      # article, server, user, comment
│   │       ├── features/      # search, auth, comments, voting, filters
│   │       ├── widgets/       # header, sidebar, feed, ticker, footer
│   │       ├── shared/        # ui-kit (shadcn), api-client, hooks, lib
│   │       └── processes/     # onboarding, checkout-like flows (2FA setup и т.п.)
│   └── api/                  # NestJS
│       └── src/
│           ├── modules/
│           │   ├── auth/          # JWT, refresh, 2FA, OAuth (Discord/Google)
│           │   ├── articles/
│           │   ├── categories/
│           │   ├── tags/
│           │   ├── comments/
│           │   ├── reports/       # жалобы
│           │   ├── users/
│           │   ├── servers/       # рейтинг серверов
│           │   ├── media/         # загрузка в S3
│           │   ├── admin/         # dashboard, roles, audit-log, backups
│           │   └── security/      # rate-limit, audit, ip-block
│           ├── common/ (guards, interceptors, pipes, filters)
│           └── prisma/
├── packages/
│   ├── ui/                    # общий UI-кит (shadcn-обёртки)
│   ├── types/                 # общие DTO/типы
│   └── config/                # eslint, tsconfig, tailwind-preset
└── infra/
    ├── docker-compose.yml     # postgres, redis, minio(s3), api, web
    └── nginx/ (reverse proxy, TLS, security headers)
```

---

## 2. Схема данных (Prisma / PostgreSQL, укрупнённо)

```prisma
model User {
  id            String   @id @default(uuid())
  email         String   @unique
  passwordHash  String
  role          Role     @default(READER)   // READER, EDITOR, MODERATOR, ADMIN
  isBanned      Boolean  @default(false)
  twoFAEnabled  Boolean  @default(false)
  twoFASecret   String?
  avatarUrl     String?
  createdAt     DateTime @default(now())
  sessions      Session[]
  comments      Comment[]
  articles      Article[] @relation("AuthorArticles")
}

model Session {
  id           String   @id @default(uuid())
  userId       String
  refreshToken String   @unique
  userAgent    String
  ip           String
  expiresAt    DateTime
  revoked      Boolean  @default(false)
}

model Article {
  id          String   @id @default(uuid())
  slug        String   @unique
  title       String
  excerpt     String
  content     String   // MDX/HTML, санитизируется при сохранении
  coverImage  String?
  status      ArticleStatus @default(DRAFT) // DRAFT, REVIEW, PUBLISHED, ARCHIVED
  hypeLevel   HypeLevel @default(RUMOR)     // CONFIRMED, LIKELY, UNCONFIRMED, RUMOR, DEBUNKED
  category    Category  @relation(fields: [categoryId], references: [id])
  categoryId  String
  tags        Tag[]
  author      User      @relation("AuthorArticles", fields: [authorId], references: [id])
  authorId    String
  views       Int      @default(0)
  publishedAt DateTime?
  seoTitle    String?
  seoDesc     String?
  comments    Comment[]
}

model Comment {
  id         String   @id @default(uuid())
  articleId  String
  userId     String
  parentId   String?     // для ответов
  content    String
  likes      Int  @default(0)
  dislikes   Int  @default(0)
  isPinned   Boolean @default(false)
  isHidden   Boolean @default(false) // после модерации
  createdAt  DateTime @default(now())
}

model Report { id String @id @default(uuid()); commentId String?; articleId String?; reason String; status ReportStatus @default(OPEN); reporterId String }

model DiscordServer {
  id         String @id @default(uuid())
  name       String
  category   String   // Игровой / RP
  membersCount Int
  rank       Int?
  trend      Int      // изменение позиции
}

model AuditLog { id String @id @default(uuid()); actorId String; action String; targetType String; targetId String; ip String; createdAt DateTime @default(now()) }

model Category { id String @id @default(uuid()); slug String @unique; name String }
model Tag      { id String @id @default(uuid()); slug String @unique; name String; articles Article[] }
```

---

## 3. API (NestJS) — основные эндпоинты

| Группа | Метод/путь | Назначение |
|---|---|---|
| Auth | `POST /auth/register` | регистрация + email-подтверждение |
| | `POST /auth/login` | логин, выдаёт access (JWT, короткий) + refresh (HttpOnly cookie) |
| | `POST /auth/refresh` | обновление access-токена |
| | `POST /auth/logout` / `/logout-all` | завершение сессии(й) |
| | `POST /auth/2fa/enable` `/2fa/verify` | TOTP 2FA |
| | `GET /auth/oauth/discord` `/oauth/google` | OAuth callback |
| Articles | `GET /articles` `?category=&tag=&sort=&page=` | лента с фильтрами и пагинацией |
| | `GET /articles/:slug` | статья + related |
| | `POST /articles` `PATCH /articles/:id` | только EDITOR/ADMIN |
| Comments | `POST /articles/:id/comments` | создать (rate-limited) |
| | `POST /comments/:id/like` `/dislike` | реакции |
| | `POST /comments/:id/report` | жалоба |
| Servers | `GET /servers/ranking` | рейтинг серверов |
| Admin | `GET /admin/stats` | Dashboard-метрики |
| | `GET/POST /admin/users` `/admin/roles` | управление ролями |
| | `GET /admin/reports` | модерация жалоб |
| | `GET /admin/audit-log` | журнал действий |
| | `POST /admin/backup` | резервная копия по требованию |
| Media | `POST /media/upload` | загрузка в S3 (валидация MIME/размера/антивирус) |
| Search | `GET /search?q=` | полнотекстовый поиск (Postgres `tsvector` + Redis-кэш подсказок) |

Все мутирующие эндпоинты защищены `JwtAuthGuard` + `RolesGuard` + `RateLimitGuard`; вход/данные — через `class-validator` DTO с whitelisting.

---

## 4. Безопасность — конкретная реализация по каждому пункту ТЗ

| Требование | Реализация |
|---|---|
| JWT + HttpOnly + Secure cookie | Access-JWT (15 мин) в памяти клиента, refresh-токен — только в `HttpOnly; Secure; SameSite=Strict` cookie |
| CSRF | Double-submit cookie токен для всех мутирующих запросов (плюс `SameSite=Strict` как основной барьер) |
| XSS / CSP | Санитизация контента статей (DOMPurify на сервере), строгий `Content-Security-Policy` без `unsafe-inline` |
| SQL Injection | Только через Prisma (параметризованные запросы), без raw SQL без валидации |
| Rate limiting / brute-force | `@nestjs/throttler` + Redis: лимиты на `/auth/login` (5/мин/IP), на комментарии и жалобы |
| Пароли | `argon2id` с индивидуальной солью |
| 2FA | TOTP (RFC 6238), резервные коды |
| Подтверждение email | Одноразовый токен со сроком действия, письмо через транзакционный email-сервис |
| DDoS на уровне приложения | Nginx/Cloudflare перед приложением + Throttler + очередь запросов через Redis |
| Аудит действий администратора | `AuditLog` на каждое admin-действие (кто, что, когда, IP) |
| Безопасная загрузка файлов | Проверка MIME по содержимому (не по расширению), лимит размера, антивирус (ClamAV в очереди обработки), приватный bucket + signed URL |
| Обработка ошибок | Единый `ExceptionFilter`, никаких стектрейсов/деталей БД в ответе клиенту |
| Контроль сессий | Таблица `Session`, страница «Активные сессии» в профиле, кнопка «Завершить все» |
| Резервные копии | Ежедневный `pg_dump` → S3, retention policy, ручной запуск из админки |

**Важно (как и указано в ТЗ):** никакая система не гарантированно «неуязвима». Список выше — набор современных best practices, минимизирующих риски, а не абсолютная гарантия.

---

## 5. Админ-панель (CMS) — структура разделов

- **Dashboard** — трафик, топ-статьи, активность за 24ч/7д/30д (графики на Recharts)
- **Статьи** — таблица со статусами (draft/review/published), WYSIWYG/MDX-редактор, SEO-поля, история версий (diff)
- **Категории / Теги** — CRUD
- **Пользователи** — поиск, роли, бан, чёрный список, блокировка по IP
- **Комментарии / Жалобы** — модерация в один клик, закрепление, скрытие
- **Медиа** — библиотека файлов с предпросмотром, статус антивирусной проверки
- **Роли** — создание Editor/Moderator/Admin с гранулярными правами (CASL-подобная модель)
- **Логи** — Audit Log с фильтрами по пользователю/действию/дате
- **Настройки сайта / SEO** — sitemap, robots.txt, OG-шаблоны, Schema.org
- **Резервные копии** — список, запуск, восстановление

---

## 6. SEO и производительность

- Next.js App Router: SSR/ISR для статей, статическая генерация категорий
- `sitemap.xml` и `robots.txt` — генерируются автоматически из БД (`next-sitemap` + кастомный источник)
- Open Graph / Twitter Cards — динамические метатеги на уровне статьи
- Schema.org `NewsArticle` / `DiscussionForumPosting` для комментариев
- ЧПУ: `/news/[slug]`, `/rumors/[slug]`
- Изображения — `next/image` + CDN (Cloudflare/S3+CloudFront), lazy loading по умолчанию
- Code splitting «из коробки» через App Router, Redis-кэш горячих ответов API (лента, рейтинг)

---

## 7. Как это развернуть дальше

1. `docker-compose up` (Postgres, Redis, MinIO как S3-совместимое хранилище)
2. `apps/api`: `prisma migrate dev`, затем реализация модулей по одному (auth → articles → comments → admin → security)
3. `apps/web`: подключение к API, перенос вёрстки из готовых `index.html` / `article.html` в React-компоненты (Feature-Sliced Design слои `entities/features/widgets`)
4. CI: линт, тесты, миграции, деплой (Vercel для web, Railway/Fly.io/VPS для api+БД)

Это естественным образом ложится в задачи для **Claude Code** — можно идти модуль за модулем (например: «реализуй `auth` модуль по этой спеке» → «теперь `articles` с пагинацией и фильтрами» и т.д.), с реальным запуском и тестированием на каждом шаге.
