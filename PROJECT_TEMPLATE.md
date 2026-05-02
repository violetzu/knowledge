# 專案架構模板

以 **Docker + Next.js + Cloudflare Tunnel** 為核心的快速部署系統。
所有服務跑在 Docker Compose 裡，透過 Cloudflare Tunnel 對外曝光，不需要開放防火牆或設定反向代理。
Next.js 同時負責 SSR、Auth、BFF；Python/FastAPI 僅在有推理或長任務需求時才加入。

**Cloudflare Tunnel 運作方式：** `tunnel` container 在 Docker 內部主動連出到 Cloudflare edge，
設定流量自動從 domain 轉進 `app:3000`，不需要公開 IP、SSL 憑證或 nginx。
docker-compose 內所有 `ports` 都被 comment 掉——對外曝光由 Tunnel 處理，
本機開發若需要直連，臨時 uncomment 即可。

每次建新專案前對照這份文件，從決策樹開始往下走，照著 checklist 填入即可。

---

## 1. 決策樹：架構怎麼選？

這份模板的預設最佳解：

- 人類使用者登入 / session / SSR 頁面 / admin / BFF 放在 Next.js
- Python / FastAPI 只做推理、分析、CV、長任務或 sidecar API

```
需要獨立 FastAPI / Python 服務嗎？
  否 → §A（Next.js + Prisma 全端，預設）
  是 → §B（Next.js + Prisma + FastAPI 服務）
```

§A：純 Web 產品、CRUD、表單、儀表板，預設先選這個
§B：主體仍是 Web 產品，但需要獨立 FastAPI 服務做推理 / 生成 / 分析

---

## 2. 環境變數（.env.example）

以下是常見基礎欄位，包含但不限於這些。
若啟用 OAuth、FastAPI service、Redis、第三方 API、額外背景服務，就在對應章節或專案內再追加。

```dotenv
# ── 登入 ──────────────────────────────────────────────────────────────────────
# 單一密碼，所有使用者共用
APP_PASSWORD=changeme

# Auth.js v5（openssl rand -base64 32）
AUTH_SECRET=
AUTH_URL=https://your-domain.example.com

# ── 資料庫 ────────────────────────────────────────────────────────────────────
# 帳號與 DB 名稱固定 = 專案名（小寫），只有密碼放 env
POSTGRES_PASSWORD=changeme

# ── Cloudflare Tunnel ─────────────────────────────────────────────────────────
CF_TUNNEL_TOKEN=

# ── LLM ───────────────────────────────────────────────────────────────────────
# 接雲端 API（OpenAI / Anthropic / 任何 OpenAI-compatible）或本地 vLLM 二選一
LLM_BASE_URL=http://localhost:8000/v1
LLM_API_KEY=
LLM_MODEL=

# ── §B FastAPI 服務（有 FastAPI service 才填）────────────────────────────────
FASTAPI_API_SECRET=changeme
INTERNAL_TOKEN_EXPIRE_MINUTES=60

# ── Dev only ──────────────────────────────────────────────────────────────────
# 讓 Tunnel HMR WebSocket 可以連到 Next.js dev server（逗號分隔多個 domain）
ALLOWED_DEV_ORIGINS=your-domain.example.com
```

> 資料庫帳密：`POSTGRES_USER` / `POSTGRES_DB` 直接 hardcode 成專案名，只有 `POSTGRES_PASSWORD` 放 env。
> 這樣 `DATABASE_URL` 只需替換密碼，不會有帳號拼錯的問題。

---

## 3. Makefile

```makefile
.PHONY: up down logs dev-up dev-down dev-logs db-backup db-restore setup help

# ── Docker ────────────────────────────────────────────────────────────────────

up:          ## Start full stack (frontend + backend + postgres)
	docker compose up -d --build

down:        ## Stop all containers
	docker compose down

logs:        ## Tail logs of all containers
	docker compose logs -f

dev-up:      ## Start full stack with frontend in dev mode
	docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d --build

dev-down:    ## Stop dev stack
	docker compose -f docker-compose.yml -f docker-compose.dev.yml down

dev-logs:    ## Tail all dev logs
	docker compose -f docker-compose.yml -f docker-compose.dev.yml logs -f

# ── Database ──────────────────────────────────────────────────────────────────

db-backup:   ## Backup database to backup.dump
	docker compose exec -T db pg_dump -U <project> -Fc <project> > backup.dump

db-restore:  ## Restore database from backup.dump
	docker compose exec -T db pg_restore -U <project> -d <project> --clean --if-exists < backup.dump

# ── Setup ─────────────────────────────────────────────────────────────────────

setup:       ## First-time setup: copy .env and start Docker
	@if [ ! -f .env ]; then \
		cp .env.example .env; \
		echo "Created .env from .env.example"; \
	else \
		echo ".env already exists, skipped"; \
	fi
	@echo "Fill in secrets in .env, then run: make up"

# ── Help ──────────────────────────────────────────────────────────────────────

help:        ## Show this help message
	@grep -E '^[a-zA-Z_-]+:.*##' $(MAKEFILE_LIST) | \
		awk 'BEGIN {FS = ":.*##"}; {printf "  \033[36m%-16s\033[0m %s\n", $$1, $$2}'

.DEFAULT_GOAL := help
```

`db-backup` / `db-restore` 的 `-U` 和 `-d` 換成專案名即可。

---

## 4. Next.js Dockerfile 基礎（§A / §B 見 A2 的 builder / runner 差異）

```dockerfile
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM deps AS dev
WORKDIR /app
EXPOSE 3000
CMD ["npm", "run", "dev", "--", "--hostname", "0.0.0.0", "--port", "3000"]

FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
EXPOSE 3000
CMD ["node", "server.js"]
```

`next.config.ts` 必須有：

```typescript
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  output: "standalone",
  ...(process.env.ALLOWED_DEV_ORIGINS && {
    allowedDevOrigins: process.env.ALLOWED_DEV_ORIGINS.split(","),
  }),
};

export default nextConfig;
```

沒有 `output: 'standalone'`，runner stage 的 `server.js` 不會生成。
`allowedDevOrigins` 讓透過 Cloudflare Tunnel 的 HMR WebSocket 不被 Next.js 擋下。

---

## 5. Auth 與授權

> Auth Pattern（Auth-1/2/3）與架構 §A/§B 是不同維度，不要混淆。

`npm install next-auth@beta`（Auth-1 / Auth-2 共用）

預設原則：

- **人類使用者登入**（login / register / session / cookie / 頁面保護）交給 Next.js
- **Python backend / FastAPI service** 負責驗證來自 Next.js proxy 的內部 token，或處理 guest / session / task 存取
- 不要把「主要登入」預設做成 frontend `localStorage` bearer token + backend 直接驗證
- 若前端只是薄殼、Python 才是系統核心，才考慮讓 FastAPI 直接持有主要登入流程

```
需要個人帳號（email/password ± OAuth）？
  是 → Auth-2（NextAuth v5 + Prisma）
  否 → Auth-1（Next.js 單一密碼）

backend 還需要辨識 user / guest / session / task 存取嗎？
  是 → 再疊加 Auth-3（backend 內部授權 / guest token）
```

---

### Auth-1：單一密碼

適用：內部工具、個人用途，`APP_PASSWORD` 全站共用，無個人帳號。
主要登入仍在 Next.js。

關鍵點：
- `trustHost: true` — Docker / Cloudflare Tunnel 環境必加，否則 callback URL 驗證失敗
- `callbacks.authorized` 回傳 `!!auth?.user` — 讓 middleware 的重新導向生效
- `middleware.ts` 仍要設定 `matcher`，排除 `/login`、`/api/`、`_next/`、`favicon.ico`，避免 redirect loop

`middleware.ts`：


```typescript
import { auth } from "@/auth";
import { NextResponse } from "next/server";

export default auth((req) => {
  if (!req.auth) {
    const url = req.nextUrl.clone();
    url.pathname = "/login";
    return NextResponse.redirect(url);
  }
  return NextResponse.next();
});

export const config = {
  matcher: ["/((?!api/|login|_next/static|_next/image|favicon.ico).*)"],
};
```

檔案結構：`src/lib/auth.ts`、`src/app/api/auth/[...nextauth]/route.ts`（re-export handlers）、`src/app/login/page.tsx`（Server Component + form action + `signIn`）

---

### Auth-2：User Accounts + OAuth

適用：公開應用，需要個人帳號。Prisma 必備（§A/§B）。

安裝：`npm install next-auth@beta @auth/prisma-adapter bcryptjs zod && npm install -D @types/bcryptjs`

`.env.example` 追加：
```dotenv
# Google OAuth（選用，不填則只有 email/password）
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
```

關鍵點：
- `session: { strategy: "jwt" }` — PrismaAdapter 預設用 database session，會多一張 Session table 且每 request 打 DB；JWT 更輕量
- `...(process.env.GOOGLE_CLIENT_ID ? [Google()] : [])` — Google provider 條件加入，不填 env 就不啟用
- `passwordHash` 欄位放在 `User` model，`bcryptjs hash(password, 12)` 儲存
- `authorize` 裡用 `z.safeParse` 驗 email + password 格式再查 DB
- 若 jwt callback 有打 DB（例如讀 isAdmin 即時同步），就用 **layout redirect** 保護路由；若 jwt callback 純讀 token 無 DB，可改用 `middleware.ts`
- `register` route 回 409 用模糊訊息，防帳號列舉攻擊
- `prisma/schema.prisma` 需加 Auth.js 標準四個 model：`User`（含 `passwordHash String?`）、`Account`、`Session`、`VerificationToken`
- `/api/auth/signin` 和 `/api/auth/register` 需加 rate limiting，防暴力破解與帳號列舉

**Rate limiting（Docker 環境，無 Redis）：** 用 in-memory LRU cache，不需要額外服務。

安裝：`npm install lru-cache`

`src/lib/rate-limit.ts`：

```typescript
import { LRUCache } from "lru-cache";

const cache = new LRUCache<string, number[]>({ max: 500 });

export function rateLimit(key: string, limit: number, windowMs: number): boolean {
  const now = Date.now();
  const timestamps = (cache.get(key) ?? []).filter((t) => now - t < windowMs);
  if (timestamps.length >= limit) return false;
  cache.set(key, [...timestamps, now]);
  return true;
}
```

在 `/api/auth/signin` 和 `/api/auth/register` route 開頭加：

```typescript
const ip = req.headers.get("cf-connecting-ip") ?? req.headers.get("x-forwarded-for") ?? "unknown";
if (!rateLimit(`auth:${ip}`, 10, 60_000)) {
  return NextResponse.json({ error: "Too many requests" }, { status: 429 });
}
```

> `cf-connecting-ip` 是 Cloudflare Tunnel 傳入的真實 IP，優先使用。
> 若之後需要跨容器共享限流狀態（多副本），才換成 Upstash Redis + `@upstash/ratelimit`。

---

### Auth-3：Backend 內部授權 / Guest / Session Token

適用：backend 需要辨識「目前是哪個使用者」、「guest 是否有權限繼續操作」、
「這個 session / task / record 是否可存取」。
這不是主要登入方案，而是疊加在 Auth-1 / Auth-2 上的 backend 授權層。

`.env.example` 追加：
```dotenv
FASTAPI_API_SECRET=changeme
INTERNAL_TOKEN_EXPIRE_MINUTES=60
```

`requirements.txt` 追加：`python-jose[cryptography]`、`passlib[bcrypt]`

關鍵點：
- backend 只信任 **Next.js proxy 以 `FASTAPI_API_SECRET` 簽發的短時效 internal JWT**，或私有網路上的可信內部 header
- 主要登入 session 放 cookie，不把 backend 主登入 token 直接交給瀏覽器
- `OAuth2PasswordBearer(auto_error=False)` — guest / optional auth 路由用，不帶 token 時回傳 `None` 而非 401
- guest flow 可用 opaque `guest_token`，放 `sessionStorage`
- backend 對 record / session / task 的最終授權判斷仍在 server 端完成
- 若真的要讓 FastAPI 直接做主要登入，視為**例外設計**，需在 `CLAUDE.md` 明寫原因

---

## 6. .gitignore 必備項目

```gitignore
.env
data/
*.dump
__pycache__/
*.pyc
.next/
node_modules/
```

---

## 7. .dockerignore

沒有 `.dockerignore`，`docker build` 會把 `node_modules/`、`.next/`、`data/` 打包進 build context，導致 cache 失效和 build 超慢。

### 根目錄 `.dockerignore`（§A / §B）

```
node_modules/
.next/
.env*
data/
__pycache__/
*.pyc
```

### `fastapi-service/.dockerignore`（§B）

根目錄的 `.dockerignore` 對 `./fastapi-service` build context 無效，需另建一份：

```
__pycache__/
*.pyc
.venv/
.env*
```

---

## 8. UI 層：Tailwind v4 + shadcn/ui

每個專案的預設 UI 套件組合。

### 安裝

```bash
# Tailwind v4（Next.js 15+ 內建支援）
npm install tailwindcss @tailwindcss/postcss @tailwindcss/typography
# shadcn/ui（會自動設定 components.json、建立 components/ui/）
npx shadcn@latest init
```

`postcss.config.mjs`：

```js
export default {
  plugins: { "@tailwindcss/postcss": {} },
};
```

### globals.css

Tailwind v4 用 `@import` + `@theme`，不用 v3 的 `@tailwind base/components/utilities`。
主色板用 `--color-brand-*` 定義，shadcn 的 CSS variable 會自動疊加在上面。

```css
@import "tailwindcss";
@plugin "@tailwindcss/typography";

@theme {
  --font-family-sans: var(--font-sans), system-ui, sans-serif;

  /* 主色板：根據專案調整這組色票 */
  --color-brand-50:  #f9fafb;
  --color-brand-100: #f3f4f6;
  --color-brand-200: #e5e7eb;
  --color-brand-300: #d1d5db;
  --color-brand-400: #9ca3af;
  --color-brand-500: #6b7280;
  --color-brand-600: #4b5563;
  --color-brand-700: #374151;
  --color-brand-800: #1f2937;
  --color-brand-900: #111827;
}

html {
  scroll-behavior: smooth;
}

body {
  background-color: var(--color-brand-50);
  color: var(--color-brand-800);
}
```

> 色板預設為中性灰階，根據專案主視覺替換整組色票即可。
> 頁面直接用 `bg-brand-50`、`text-brand-800` 等 class，不需要額外設定。

### 使用 shadcn 元件

```bash
npx shadcn@latest add button card input
```

元件放在 `src/components/ui/`，直接 import 使用，不需要額外設定。

---

## 9. §A — Next.js + Prisma 全端（預設）

### A1. Prisma 設定

```
npm install prisma @prisma/client
npx prisma init --datasource-provider postgresql
```

`prisma/schema.prisma` 的 `datasource`：

```prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}
```

`DATABASE_URL` 格式：
```
postgresql://<project>:${POSTGRES_PASSWORD}@db:5432/<project>
```

docker-compose.yml 的 `app.environment` 直接傳入即可。

### A2. entrypoint.sh（自動 migration）

容器啟動時自動跑 migration，不需要手動介入。

`entrypoint.sh`（放根目錄）：

```sh
#!/bin/sh
set -eu
npx prisma migrate deploy
exec node server.js
```

> **注意**：單一副本沒問題。若未來需要多副本水平擴展，需改用外部 migration job（K8s initContainer 或 CI step），避免多容器同時執行 migrate 的 race condition。

Dockerfile builder stage 需加 `npx prisma generate`（在 `npm run build` 之前），runner stage 需複製 prisma 相關檔案：

```dockerfile
# builder stage（替換 section 4 的 builder stage）
FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npx prisma generate && npm run build

# runner stage（替換 section 4 的 runner stage）
FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
COPY --from=builder /app/prisma ./prisma
COPY --from=builder /app/node_modules/.prisma ./node_modules/.prisma
COPY --from=builder /app/node_modules/prisma ./node_modules/prisma
COPY --from=builder /app/node_modules/@prisma/engines ./node_modules/@prisma/engines
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
EXPOSE 3000
CMD ["/entrypoint.sh"]
```

### A3. 目錄結構（§A）

Next.js 直接放根目錄（無 `frontend/` 子目錄）。

```
<project>/
├── Makefile
├── docker-compose.yml
├── docker-compose.dev.yml
├── .env
├── .env.example
├── .gitignore
├── Dockerfile          # Next.js 多階段建置（見 A2 的 runner stage 差異）
├── entrypoint.sh       # migration + 啟動
├── next.config.ts      # output: 'standalone'
├── package.json
├── prisma/
│   └── schema.prisma
├── src/
│   ├── app/
│   │   ├── actions.ts   # Server Actions
│   │   └── ...
│   └── lib/
│       └── prisma.ts    # PrismaClient singleton
└── data/
    └── postgres/
```

`lib/prisma.ts` singleton（避免 dev HMR 建多個 connection）：

```typescript
import { PrismaClient } from "@prisma/client";

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient };

export const prisma =
  globalForPrisma.prisma ?? new PrismaClient();

if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma;
```

### A4. docker-compose.yml（§A）

`context: .`（根目錄），無 backend service，service 名稱為 `app`。

```yaml
x-logging: &default-logging
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"

services:
  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      TZ: Asia/Taipei
      POSTGRES_USER: <project>
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: <project>
    volumes:
      - ./data/postgres:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U <project>"]
      interval: 5s
      timeout: 5s
      retries: 5
    logging: *default-logging

  app:
    build:
      context: .
      dockerfile: Dockerfile
    restart: unless-stopped
    environment:
      TZ: Asia/Taipei
      AUTH_URL: ${AUTH_URL}
      AUTH_SECRET: ${AUTH_SECRET}
      AUTH_TRUST_HOST: "true"
      APP_PASSWORD: ${APP_PASSWORD}
      DATABASE_URL: postgresql://<project>:${POSTGRES_PASSWORD}@db:5432/<project>
    depends_on:
      db:
        condition: service_healthy
    logging: *default-logging

  tunnel:
    image: cloudflare/cloudflared:latest
    restart: unless-stopped
    command: tunnel --no-autoupdate run --token ${CF_TUNNEL_TOKEN}
    depends_on:
      - app
    logging: *default-logging
```

### A5. docker-compose.dev.yml（§A）

service 名稱為 `app`，mount 根目錄（`.:/app`），dev command 額外跑 `prisma generate`。

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: dev
    command: >
      sh -c "if ! diff -q package-lock.json node_modules/.package-lock.json > /dev/null 2>&1; then npm ci; fi
             && npx prisma generate
             && npm run dev -- --hostname 0.0.0.0 --port 3000"
    environment:
      AUTH_URL: ${AUTH_URL}
      AUTH_SECRET: ${AUTH_SECRET}
      AUTH_TRUST_HOST: "true"
      APP_PASSWORD: ${APP_PASSWORD}
      DATABASE_URL: postgresql://<project>:${POSTGRES_PASSWORD}@db:5432/<project>
      WATCHPACK_POLLING: "true"
      CHOKIDAR_USEPOLLING: "true"
      ALLOWED_DEV_ORIGINS: ${ALLOWED_DEV_ORIGINS:-}
    volumes:
      - .:/app
      - app_node_modules:/app/node_modules
      - app_next_cache:/app/.next
    # ports:
    #   - "3000:3000"

volumes:
  app_node_modules:
  app_next_cache:
```

---

## 10. §B — Next.js + Prisma + FastAPI 服務（需要 Python 時的預設最佳解）

主應用是全端 Next.js，但需要一個獨立 FastAPI 服務（推理、生成、CV、背景任務、自訂 API 等）。
Next.js 透過 proxy route 呼叫 FastAPI 服務，不直接暴露後端 port。

### B1. FastAPI 服務 Dockerfile

```dockerfile
FROM python:3.11-slim

COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

WORKDIR /app

COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

COPY . .
CMD ["uv", "run", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### B2. Next.js → FastAPI 服務 Proxy Route

`src/app/api/fastapi/[...path]/route.ts`：

```typescript
import { NextRequest, NextResponse } from "next/server";

const FASTAPI_SERVICE_URL = process.env.FASTAPI_SERVICE_URL ?? "http://localhost:8000";

async function handler(req: NextRequest, { params }: { params: Promise<{ path: string[] }> }) {
  const { path } = await params;
  const url = `${FASTAPI_SERVICE_URL}/${path.join("/")}${req.nextUrl.search}`;

  const headers = new Headers(req.headers);
  headers.delete("host");

  const res = await fetch(url, {
    method: req.method,
    headers,
    body: ["GET", "HEAD"].includes(req.method) ? undefined : req.body,
    duplex: "half",
  } as RequestInit);

  return new NextResponse(res.body, {
    status: res.status,
    headers: res.headers,
  });
}

export const GET = handler;
export const POST = handler;
export const PUT = handler;
export const PATCH = handler;
export const DELETE = handler;
export const HEAD = handler;
export const OPTIONS = handler;
```

前端呼叫一律用 `/api/fastapi/...`，不直接打 `FASTAPI_SERVICE_URL`。

若 FastAPI service 或 backend 需要辨識登入使用者，由 Next.js server 端依 session 產生
短時效 internal token 再轉發，不要讓瀏覽器直接持有 Python 服務的主要登入 token。

### B3. docker-compose.yml（§B）

```yaml
x-logging: &default-logging
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"

services:
  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      TZ: Asia/Taipei
      POSTGRES_USER: <project>
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: <project>
    volumes:
      - ./data/postgres:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U <project>"]
      interval: 5s
      timeout: 5s
      retries: 5
    logging: *default-logging

  fastapi:
    build:
      context: ./fastapi-service
      dockerfile: Dockerfile
    restart: unless-stopped
    environment:
      TZ: Asia/Taipei
      FASTAPI_API_SECRET: ${FASTAPI_API_SECRET}
    volumes:
      - ./data/fastapi:/data
      - ./data/hf-cache:/root/.cache/huggingface
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"]
      interval: 30s
      timeout: 10s
      retries: 10
      start_period: 60s
    logging: *default-logging

  app:
    build:
      context: .
      dockerfile: Dockerfile
    restart: unless-stopped
    environment:
      TZ: Asia/Taipei
      AUTH_URL: ${AUTH_URL}
      AUTH_SECRET: ${AUTH_SECRET}
      AUTH_TRUST_HOST: "true"
      APP_PASSWORD: ${APP_PASSWORD}
      DATABASE_URL: postgresql://<project>:${POSTGRES_PASSWORD}@db:5432/<project>
      FASTAPI_SERVICE_URL: http://fastapi:8000
      FASTAPI_API_SECRET: ${FASTAPI_API_SECRET}
    depends_on:
      db:
        condition: service_healthy
      fastapi:
        condition: service_healthy
    logging: *default-logging

  tunnel:
    image: cloudflare/cloudflared:latest
    restart: unless-stopped
    command: tunnel --no-autoupdate run --token ${CF_TUNNEL_TOKEN}
    depends_on:
      - app
    logging: *default-logging
```

### B4. docker-compose.dev.yml（§B）

`app` service 用 dev target + volume mount；`fastapi` 通常無 hot reload 必要，掛 source volume 加 `--reload` 即可。

```yaml
services:
  fastapi:
    volumes:
      - ./fastapi-service:/app
    command: uv run uvicorn main:app --host 0.0.0.0 --port 8000 --reload

  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: dev
    command: >
      sh -c "if ! diff -q package-lock.json node_modules/.package-lock.json > /dev/null 2>&1; then npm ci; fi
             && npx prisma generate
             && npm run dev -- --hostname 0.0.0.0 --port 3000"
    environment:
      AUTH_URL: ${AUTH_URL}
      AUTH_SECRET: ${AUTH_SECRET}
      AUTH_TRUST_HOST: "true"
      APP_PASSWORD: ${APP_PASSWORD}
      DATABASE_URL: postgresql://<project>:${POSTGRES_PASSWORD}@db:5432/<project>
      FASTAPI_SERVICE_URL: http://fastapi:8000
      FASTAPI_API_SECRET: ${FASTAPI_API_SECRET}
      WATCHPACK_POLLING: "true"
      CHOKIDAR_USEPOLLING: "true"
      ALLOWED_DEV_ORIGINS: ${ALLOWED_DEV_ORIGINS:-}
    volumes:
      - .:/app
      - app_node_modules:/app/node_modules
      - app_next_cache:/app/.next
    # ports:
    #   - "3000:3000"

volumes:
  app_node_modules:
  app_next_cache:
```

### B5. 目錄結構（§B）

```
<project>/
├── Makefile
├── docker-compose.yml
├── docker-compose.dev.yml
├── .env
├── .env.example
├── .gitignore
├── Dockerfile          # Next.js（同 §A）
├── entrypoint.sh       # migration + 啟動
├── next.config.ts      # output: 'standalone'
├── package.json
├── prisma/
│   └── schema.prisma
├── src/
│   ├── app/
│   │   ├── api/fastapi/[...path]/route.ts   # proxy → FastAPI 服務
│   │   ├── actions.ts
│   │   └── ...
│   └── lib/
│       └── prisma.ts
├── fastapi-service/    # Python FastAPI 服務
│   ├── Dockerfile
│   ├── pyproject.toml
│   ├── uv.lock
│   └── main.py
└── data/
    ├── postgres/
    ├── fastapi/
    └── hf-cache/
```


## 11. §D — 附加服務（按需複製）

按需複製貼上，不一定每個專案都需要。

### D0. FastAPI CV / 影音依賴

需要 OpenCV 或 ffmpeg 時，在 `fastapi-service/Dockerfile` 的 `WORKDIR /app` 前加：

```dockerfile
RUN apt-get update && \
    apt-get install -y --no-install-recommends ffmpeg libgl1 libglib2.0-0 && \
    rm -rf /var/lib/apt/lists/*
```

### D1. pgvector（PostgreSQL + 向量索引）

替換 docker-compose 的 `db` service image：

```yaml
  db:
    image: pgvector/pgvector:pg16   # 替換 postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: <project>
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: <project>
    volumes:
      - ./data/postgres:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U <project>"]
      interval: 5s
      timeout: 5s
      retries: 5
```

在 Prisma schema 啟用 extension：

```prisma
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["postgresqlExtensions"]
}

datasource db {
  provider   = "postgresql"
  url        = env("DATABASE_URL")
  extensions = [vector]
}
```

### D2. vLLM（本地推理，預設不啟動）

加到 docker-compose，用 Docker profile 讓 `make up` 不啟動、`make up-vllm` 才啟動：

```yaml
  vllm:
    image: vllm/vllm-openai:latest
    profiles: [vllm]
    command: >
      --model <model-name>
      --dtype bfloat16
      --max-model-len 8192
      --port 8000
    # ports:
    #   - "8000:8000"
    volumes:
      - ./data/vllm-cache:/root/.cache/huggingface
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    restart: unless-stopped

  # 獨立 embedding 服務（選用）
  embedding:
    image: vllm/vllm-openai:latest
    profiles: [vllm]
    command: >
      --model BAAI/bge-m3
      --task embed
      --dtype float16
      --port 8001
    # ports:
    #   - "8001:8001"
    volumes:
      - ./data/embedding-cache:/root/.cache/huggingface
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    restart: unless-stopped
```

Makefile 加一個 target：

```makefile
up-vllm:     ## Start full stack with local vLLM
	docker compose --profile vllm up -d --build
```

vLLM 啟動後，`LLM_BASE_URL` 指向 `http://vllm:8000/v1`（docker 內部）或 `http://localhost:8000/v1`（本機）。

---

## 12. 首次建立 Checklist

```
□ 複製 Makefile，替換 db-backup / db-restore 裡的專案名
□ 複製 .env.example，填入實際值
□ 決定 §A / §B
□ 複製 docker-compose.yml，搜尋替換 <project> → 實際專案名
   §A：用 A4 的模板（context: .，無獨立 backend）
   §B：用 B3 的模板（含 fastapi service block）
□ 複製 docker-compose.dev.yml（§A 用 A5；§B 用 B4）
□ 建立根目錄 .dockerignore（section 7）
□ Dockerfile 放根目錄（含 A2 的 prisma runner stage），next.config.ts 用 section 4 模板
□ init Prisma，建 lib/prisma.ts singleton，建 entrypoint.sh
□ §B：建 fastapi-service/ 目錄和 api/fastapi/[...path]/route.ts proxy
□ §B：在 fastapi-service/ 建 .dockerignore（section 7）
□ 決定 Auth Pattern（section 5）：Auth-1 單一密碼 / Auth-2 個人帳號
   Auth-1/2：建 lib/auth.ts、api/auth/[...nextauth]/route.ts
   Auth-1：建 middleware.ts
   Auth-2：schema.prisma 加 Auth.js 模型、建 api/auth/register/route.ts、layout redirect
□ 若 backend 需要 user / guest / session / task 授權，再疊加 Auth-3
   Auth-3：建 backend/auth.py 或對應授權模組、proxy 內部 token、guest/session 存取規則
□ .gitignore 加 data/ 和 .env
□ 填 CLAUDE.md、AGENTS.md（section 13 / 14）
□ make setup    ← 複製 .env，填完 secrets
□ make up       ← 確認所有 container 都 healthy
□ make dev-up   ← 確認 HMR 在 Tunnel 下可用
```

---

## 13. CLAUDE.md 模板

```markdown
# CLAUDE.md

@AGENTS.md
```

---

## 14. AGENTS.md 模板

每個專案複製後填入實際值。括號內的選項擇一保留，其餘刪除。

```markdown
# AGENTS.md

# <project>

## 架構

（擇一）§A：Next.js + Prisma 全端，無獨立 backend
（擇一）§B：Next.js + Prisma 全端 + FastAPI 服務（`fastapi-service/`）

專案名（DB user / DB name）：`<project>`

## Auth

（擇一）Auth-1：Next.js 單一密碼（`APP_PASSWORD`），middleware.ts 保護全站
（擇一）Auth-2：Next.js 個人帳號，email/password（+ Google OAuth），layout redirect 保護
（選配）Auth-3：backend 內部授權 / guest / session token；不是主要登入

## 常用指令

| 指令 | 用途 |
|---|---|
| `make up` | 啟動生產 stack |
| `make dev-up` | 啟動 dev stack（Next.js HMR）|
| `make dev-logs` | tail 所有 container log |
| `make db-backup` | 備份 DB |

## 規則

- §B：前端呼叫 FastAPI 一律走 `/api/fastapi/...`，不直接用 `FASTAPI_SERVICE_URL`
- 人類使用者登入（Auth-1/2）預設放在 Next.js；backend 只處理內部 token 與資源授權（Auth-3）
- DB migration：本地用 `npx prisma migrate dev` 建立 migration 檔案，部署時 `entrypoint.sh` 自動執行 `prisma migrate deploy`
- （專案自訂規則加在這裡）

## 環境

- Node 20，Python 3.11
- PostgreSQL 16，Prisma
```
