# 專案架構模板 v2026.05.07（Next.js 16 + Prisma 7）

> 給 AI 的閱讀說明：只記錄非顯而易見的決策與陷阱。標準語法 AI 自行判斷，本文僅在有踩坑風險時才列出完整範例。
> **本機不安裝任何 Node / Python 套件**——所有指令透過 Docker 執行。

---

## 0. 建立新專案前：必要資訊確認

**使用者提供這份模板來建立新專案時，在動手之前必須先確認以下資訊。未提供的項目一律開口問，不要自行假設。**

### 必問（沒有就無法開始）

| 項目 | 說明 |
|------|------|
| **專案名稱** | 全小寫英數，會用作 DB user、DB name、資料夾名、docker service 名 |
| **架構** | §A / §B / §C（見 Section 1 決策樹） |
| **Auth 方式** | Auth-0 不需要 / Auth-1 單一密碼 / Auth-2 個人帳號 |
| **部署網址** | `https://<domain>`，填入 AGENTS.md；Auth-1/2 另填入 `AUTH_URL` |

### 選問（影響架構，沒提到就問）

| 項目 | 說明 |
|------|------|
| **§B/C：FastAPI 負責什麼** | 推理 / CV / 長任務 / 其他——決定 Dockerfile base image 和 GPU 設定 |
| **需要 GPU 嗎** | 是 → 哪個 service 用；否 → 跳過 GPU 相關設定 |
| **需要 LLM / embedding 嗎** | 是 → 本地 vLLM 還是雲端 API（影響 §D2 和 env 變數） |
| **需要 pgvector 嗎** | 影響 DB image 和 Prisma schema |
| **Auth-2：需要 Google OAuth 嗎** | 影響 env 變數和 provider 設定 |

### 確認後才開始

收到以上資訊後，依序產出：
1. `AGENTS.md`（填好所有決策）
2. `.env.example`
3. `docker-compose.yml` / `docker-compose.dev.yml`
4. `Dockerfile`（± `fastapi-service/Dockerfile`）
5. `Makefile`、`entrypoint.sh`、`next.config.ts`、`prisma.config.ts`（§A/B 且有 Prisma 時）
6. `CLAUDE.md`（內容固定只有一行：`@AGENTS.md`）

---

以 **Docker + Next.js 16 + Prisma 7 + Cloudflare Tunnel** 為核心的快速部署系統。
所有服務跑在 Docker Compose 裡，透過 Cloudflare Tunnel 對外曝光，不需要開放防火牆或設定反向代理。
Next.js 同時負責 SSR、Auth、BFF；Python/FastAPI 僅在有推理或長任務需求時才加入。

**版本基準：**
- Node image：`node:22-bookworm-slim`
- Next.js 16
- Prisma 7（v7 專用，不保留 v6 寫法）
- PostgreSQL 16

**Cloudflare Tunnel：** `tunnel` container 主動連出到 Cloudflare edge，流量從 domain 轉進 `app:3000`，
不需要公開 IP、SSL 憑證或 nginx。

**Port 規則：**
- `docker-compose.yml`（production）：**所有 service 的 `ports` 都不寫**，包含 DB、cache、backend——對外由 Tunnel，container 間走 Docker 內網
- `docker-compose.dev.yml`（dev overlay）：可選擇性暴露 `app:3000` 方便本機直連，其他 service 仍不暴露

---

## 1. 決策樹：架構怎麼選？

```
需要獨立 FastAPI / Python 服務嗎？
  否 → §A（Next.js 全端，預設）
  是 → Python 只是 sidecar，Next.js 仍是主體？
    是 → §B（Next.js 主導 + FastAPI sidecar）
    否 → §C（FastAPI 主導 + Next.js 薄殼）
```

- §A：純 Web 產品、CRUD、表單、儀表板，Next.js 直接操作 DB（Prisma）
- §B：主體仍是 Next.js，Python 負責推理 / 生成 / CV / 長任務，DB 由 Next.js 管
- §C：Python 是系統核心，自己管 DB 和業務邏輯；Next.js 只負責 UI / Auth / 轉發，不碰 DB
- 人類使用者登入 / session / SSR / admin / BFF 一律放 Next.js，不放 FastAPI

---

## 2. 版本差異與關鍵規則

**Next.js 16：**
- `next lint` 已移除，lint 改用 ESLint CLI；dev dependency 加 `eslint eslint-config-next`
- `next.config.ts` 不放 `eslint` option

**Prisma 7（ESM 強制）：**
- `package.json` 必須有 `"type": "module"`
- generator provider 改 `prisma-client`（非 `prisma-client-js`），`output` 必填
- datasource 不放 `url`，改放 `prisma.config.ts`
- `defineConfig`、`env` 從 `"prisma/config"` import
- `PrismaClient` 從 `@/generated/prisma/client` import，**不從 `@prisma/client`**
- driver adapter 必須：`@prisma/adapter-pg` + `pg`（runtime），`prisma` + `@types/pg`（dev）
- `.gitignore` 加 `src/generated/prisma/`（generated client 不提交）

**next-auth@beta（v5）：**
- 仍是 beta 標籤，API 穩定可正常使用
- `AUTH_TRUST_HOST: "true"` 需同時放在 docker-compose `environment`（env var 形式驅動，不只靠 config 的 `trustHost: true`）

**Server Actions vs API Route（唯一差別）：**
- streaming（LLM / SSE）和外部進入點（webhook）才用 API Route；其餘一律 Server Actions

---

## 3. 環境變數（.env.example）

**Secret 生成方式：**
```bash
openssl rand -base64 32   # AUTH_SECRET、FASTAPI_API_SECRET 等
```

`.env.example` 依實際決策組合，不適用的變數一律不加。

**通用（所有專案）：**
```dotenv
CF_TUNNEL_TOKEN=
ALLOWED_DEV_ORIGINS=your-domain.example.com   # Tunnel HMR WebSocket 用
```

**有 PostgreSQL 時加入：**
```dotenv
POSTGRES_PASSWORD=CHANGE_ME_DB_PASSWORD
# POSTGRES_USER / POSTGRES_DB hardcode 成專案名，只有密碼放 env，避免帳號拼錯
```

**Auth-1 / Auth-2 才加入：**
```dotenv
AUTH_SECRET=CHANGE_ME_RUN_openssl_rand_base64_32
AUTH_URL=https://your-domain.example.com
```

**Auth-1 才加入：**
```dotenv
APP_PASSWORD=CHANGE_ME_APP_PASSWORD
```

**Auth-2 + Google OAuth 才加入：**
```dotenv
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
```

**§B / §C 有 FastAPI 才加入：**
```dotenv
FASTAPI_API_SECRET=CHANGE_ME_RUN_openssl_rand_base64_32
INTERNAL_TOKEN_EXPIRE_MINUTES=60
```

**LLM / embedding 才加入：**
```dotenv
LLM_BASE_URL=http://localhost:8000/v1
LLM_API_KEY=
LLM_MODEL=
```

> 所有 secret 欄位一律用 `CHANGE_ME_...` 前綴，不用 `changeme` 這種弱值。

---

## 4. Makefile

`<project>` 替換成專案名（兩處：`db-backup` / `db-restore` 的 pg 指令）。無 DB 時移除這兩個 target。

```makefile
.PHONY: up down logs dev-up dev-down dev-logs db-backup db-restore setup help

up:           ## Start full stack (frontend + backend + postgres)
	docker compose up -d --build

down:         ## Stop all containers
	docker compose down

logs:         ## Tail logs of all containers
	docker compose logs -f

dev-up:       ## Start full stack with frontend in dev mode
	docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d --build

dev-down:     ## Stop dev stack
	docker compose -f docker-compose.yml -f docker-compose.dev.yml down

dev-logs:     ## Tail all dev logs
	docker compose -f docker-compose.yml -f docker-compose.dev.yml logs -f

db-backup:    ## Backup database to backup.dump
	docker compose exec -T db pg_dump -U <project> -Fc <project> > backup.dump

db-restore:   ## Restore database from backup.dump
	docker compose exec -T db pg_restore -U <project> -d <project> --clean --if-exists < backup.dump

setup:        ## First-time setup: copy .env
	@if [ ! -f .env ]; then cp .env.example .env; echo "Created .env"; else echo ".env exists, skipped"; fi

help:         ## Show this help message
	@grep -E '^[a-zA-Z_-]+:.*##' $(MAKEFILE_LIST) | \
		awk 'BEGIN {FS = ":.*##"}; {printf "  \033[36m%-16s\033[0m %s\n", $$1, $$2}'

.DEFAULT_GOAL := help
```

---

## 5. next.config.ts 必填項目

```typescript
import type { NextConfig } from "next";

const securityHeaders = [
  { key: "X-Frame-Options", value: "DENY" },
  { key: "X-Content-Type-Options", value: "nosniff" },
  { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
  { key: "Permissions-Policy", value: "camera=(), microphone=(), geolocation=()" },
];

const nextConfig: NextConfig = {
  output: "standalone",           // 缺少時 runner stage 的 server.js 不會生成
  ...(process.env.ALLOWED_DEV_ORIGINS && {
    allowedDevOrigins: process.env.ALLOWED_DEV_ORIGINS.split(","),
  }),                             // Cloudflare Tunnel HMR WebSocket；必須用 env，不能 hardcode
  async headers() {
    if (process.env.NODE_ENV !== "production") return [];
    return [{ source: "/(.*)", headers: securityHeaders }];
  },
};

export default nextConfig;
```

---

## 6. Dockerfile

### 基礎版（無 Prisma，§C 的 Next.js 端使用）

```dockerfile
FROM node:22-bookworm-slim AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci

FROM deps AS dev
WORKDIR /app
EXPOSE 3000
CMD ["npm", "run", "dev", "--", "--hostname", "0.0.0.0", "--port", "3000"]

FROM node:22-bookworm-slim AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:22-bookworm-slim AS runner
WORKDIR /app
ENV NODE_ENV=production
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
EXPOSE 3000
CMD ["node", "server.js"]
```

### 有 Prisma 時（§A/B）的差異

builder stage：
```dockerfile
RUN npx prisma generate && npm run build
```

runner stage 額外複製（standalone trace 不包含這些）並替換 CMD：
```dockerfile
COPY --from=builder /app/prisma ./prisma
COPY --from=builder /app/package*.json ./
COPY --from=builder /app/node_modules/prisma ./node_modules/prisma
COPY --from=builder /app/node_modules/@prisma ./node_modules/@prisma
COPY --from=builder /app/node_modules/dotenv ./node_modules/dotenv
COPY --from=builder /app/prisma.config.ts ./prisma.config.ts
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
CMD ["node", "server.js"]
```

**entrypoint.sh（production migration 自動執行）：**
```sh
#!/bin/sh
set -eu
npx prisma migrate deploy
exec "$@"
```

dev stack 使用 `target: dev` 與自己的 `command`，不經過 production entrypoint。

### FastAPI Dockerfile（§B/C 基礎版）

```dockerfile
FROM python:3.11-slim
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev
COPY . .
CMD ["uv", "run", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

`main.py` 至少需要 healthcheck：
```python
from fastapi import FastAPI
app = FastAPI()

@app.get("/health")
def health():
    return {"ok": True}
```

---

## 7. docker-compose.yml

**Port 規則：所有 service 不寫 `ports`**，包含 DB、FastAPI——對外只走 Cloudflare Tunnel，container 間走 Docker 內網。

### §A 完整版

```yaml
x-logging: &default-logging
  driver: json-file
  options:
    max-size: "10m"
    max-file: "3"

services:
  db:
    image: postgres:16
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

### §B 在 §A 基礎上的差異

新增 `fastapi` service：
```yaml
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
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"]
      interval: 30s
      timeout: 10s
      retries: 10
      start_period: 60s
    logging: *default-logging
```

`app` service 追加（`depends_on` 要同時保留 `db`，不要漏）：
```yaml
    environment:
      FASTAPI_SERVICE_URL: http://fastapi:8000
      FASTAPI_API_SECRET: ${FASTAPI_API_SECRET}
    depends_on:
      db:
        condition: service_healthy
      fastapi:
        condition: service_healthy
```

### §C 與 §B 的差異

- `app` 移除 `DATABASE_URL`，移除 `depends_on.db`，只等 `fastapi`
- `fastapi` 新增 `DATABASE_URL` 並 `depends_on` db

```yaml
  fastapi:
    environment:
      TZ: Asia/Taipei
      DATABASE_URL: postgresql://<project>:${POSTGRES_PASSWORD}@db:5432/<project>
      FASTAPI_API_SECRET: ${FASTAPI_API_SECRET}
    depends_on:
      db:
        condition: service_healthy

  app:
    environment:
      TZ: Asia/Taipei
      AUTH_URL: ${AUTH_URL}
      AUTH_SECRET: ${AUTH_SECRET}
      AUTH_TRUST_HOST: "true"
      APP_PASSWORD: ${APP_PASSWORD}
      FASTAPI_SERVICE_URL: http://fastapi:8000
      FASTAPI_API_SECRET: ${FASTAPI_API_SECRET}
    depends_on:
      fastapi:
        condition: service_healthy
```

---

## 8. docker-compose.dev.yml

### §A 完整版

```yaml
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: dev
    command: >
      sh -c "[ ! -f node_modules/.package-lock.json ] ||
             diff -q package-lock.json node_modules/.package-lock.json > /dev/null 2>&1 ||
             npm ci
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
    #   - "3000:3000"  # 需要本機直連時 uncomment，平時走 Tunnel

volumes:
  app_node_modules:
  app_next_cache:
```

> `WATCHPACK_POLLING` / `CHOKIDAR_USEPOLLING`：Docker volume mount 下 inotify 不可靠，需改輪詢才能觸發 HMR。

### §B/C 追加 fastapi dev override

```yaml
services:
  fastapi:
    volumes:
      - ./fastapi-service:/app
    command: uv run uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

§C 的 `app` command 移除 `npx prisma generate`；`--reload` 時 alembic migration 不自動跑，開發期手動執行：
```bash
docker compose run --rm fastapi alembic upgrade head
```

---

## 9. §A — Prisma 設定

```prisma
# prisma/schema.prisma
generator client {
  provider = "prisma-client"          # 不是 prisma-client-js
  output   = "../src/generated/prisma"
}

datasource db {
  provider = "postgresql"
  # 不放 url，url 在 prisma.config.ts
}
```

```typescript
// prisma.config.ts（根目錄）
import "dotenv/config";
import { defineConfig, env } from "prisma/config";

export default defineConfig({
  schema: "prisma/schema.prisma",
  migrations: { path: "prisma/migrations" },
  datasource: { url: env("DATABASE_URL") },
});
```

`DATABASE_URL` 格式：`postgresql://<project>:${POSTGRES_PASSWORD}@db:5432/<project>`

```typescript
// src/lib/prisma.ts — singleton（dev HMR 下防止多重連線）
import { PrismaPg } from "@prisma/adapter-pg";
import { PrismaClient } from "@/generated/prisma/client";  // 不從 @prisma/client

const globalForPrisma = globalThis as unknown as { prisma?: PrismaClient };
const adapter = new PrismaPg({ connectionString: process.env.DATABASE_URL! });
export const prisma = globalForPrisma.prisma ?? new PrismaClient({ adapter });
if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma;
```

---

## 10. §B / §C 差異對照

| 項目 | §B（Next.js 主導） | §C（FastAPI 主導） |
|------|----|----|
| Next.js Prisma | 有 | 無 |
| DB 管理 | Prisma（Next.js） | SQLModel + Alembic（FastAPI） |
| entrypoint | Next.js 側（`prisma migrate deploy`） | FastAPI 側（`alembic upgrade head`） |
| `app.depends_on` | `db` + `fastapi` | `fastapi` only |
| `DATABASE_URL` | `app` service | `fastapi` service |
| dev command | 含 `prisma generate` | 不含 |
| Next.js Dockerfile | Section 6 Prisma 版 | Section 6 無 Prisma 基礎版 |

**§C 的 FastAPI DB 結構：**
```
fastapi-service/
├── alembic/env.py + versions/
├── alembic.ini
├── models.py
├── database.py
└── main.py
```

FastAPI entrypoint.sh（§C）：
```sh
#!/bin/sh
set -eu
alembic upgrade head
exec "$@"
```

**§B/C 共用規則：**
- 前端呼叫 FastAPI 一律走 `src/app/api/fastapi/[...path]/route.ts` proxy，不直接打 `FASTAPI_SERVICE_URL`
- FastAPI 不暴露 `ports`

proxy route 骨架（Next.js 15+ params 為 Promise）：
```typescript
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
  return new NextResponse(res.body, { status: res.status, headers: res.headers });
}
```

---

## 11. 目錄結構

### §A

```
<project>/
├── prisma.config.ts                # Prisma 7 設定，放根目錄（不在 prisma/ 內）
├── entrypoint.sh                   # production migration 用
├── prisma/schema.prisma
├── src/
│   ├── generated/prisma/           # generated client，不提交（.gitignore 排除）
│   └── lib/prisma.ts
└── data/postgres/                  # volume 掛載目錄，不提交
```

### §B/C 追加

```
├── fastapi-service/
│   ├── Dockerfile
│   ├── .dockerignore               # 獨立一份，根目錄的對子目錄 build context 無效
│   ├── pyproject.toml
│   ├── uv.lock
│   ├── main.py
│   └── entrypoint.sh              # §C only（alembic upgrade head）
└── data/
    ├── postgres/
    ├── fastapi/
    └── hf-cache/                  # HuggingFace model cache（有 ML 模型時）
```

---

## 12. Auth

> Auth Pattern（Auth-0/1/2）與架構 §A/§B/§C 是不同維度。

### Auth-0：無登入

不建立 `auth.ts`、`middleware.ts`、login page、`/api/auth/[...nextauth]`，不加任何 Auth env 變數。

### Auth-1：單一密碼

```typescript
// src/lib/auth.ts
import NextAuth from "next-auth";
import Credentials from "next-auth/providers/credentials";

export const { auth, handlers, signIn, signOut } = NextAuth({
  trustHost: true,
  providers: [
    Credentials({
      credentials: { password: { type: "password" } },
      async authorize({ password }) {
        if (password !== process.env.APP_PASSWORD) return null;
        return { id: "admin", name: "Admin" };
      },
    }),
  ],
  callbacks: {
    authorized: ({ auth }) => !!auth?.user,
  },
});
```

```typescript
// src/middleware.ts
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

`/api/auth/[...nextauth]/route.ts` re-export handlers；docker-compose `environment` 加 `AUTH_TRUST_HOST: "true"`。

### Auth-2：個人帳號

套件：`next-auth@beta @auth/prisma-adapter bcryptjs zod`，dev: `@types/bcryptjs`

關鍵點：
- `session: { strategy: "jwt" }` — PrismaAdapter 預設 database session 每 request 打 DB，JWT 更輕量
- jwt callback 不要加 DB 查詢；若加了（例如讀取 role），改用 **layout redirect** 保護路由（middleware 只讀 JWT 不打 DB，callback 有 DB 查詢時 middleware 驗 session 會有問題）
- `passwordHash` 放在 `User` model，`bcryptjs.hash(password, 12)`
- Google provider 條件加入：`...(process.env.GOOGLE_CLIENT_ID ? [Google()] : [])`
- register route 409 用模糊訊息（防帳號列舉攻擊）
- Auth.js 四個標準 model：`User`（含 `passwordHash String?`）、`Account`、`Session`、`VerificationToken`
- signin / register route 加 rate limiting

Rate limiting（無 Redis，in-memory LRU）：
```typescript
// src/lib/rate-limit.ts
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

```typescript
// route 開頭
const ip = req.headers.get("cf-connecting-ip") ?? req.headers.get("x-forwarded-for") ?? "unknown";
if (!rateLimit(`auth:${ip}`, 10, 60_000)) {
  return NextResponse.json({ error: "Too many requests" }, { status: 429 });
}
```

`cf-connecting-ip` 是 Cloudflare Tunnel 傳入的真實 IP，優先使用。

### Auth-3：Backend 內部授權（疊加在 Auth-1/2 上）

Next.js 簽發給 FastAPI 的 short-lived internal JWT（`FASTAPI_API_SECRET` 簽名，`INTERNAL_TOKEN_EXPIRE_MINUTES` 控制時效）：

```typescript
// src/lib/internal-token.ts
import { SignJWT } from "jose";

export async function createInternalToken(userId: string) {
  const secret = new TextEncoder().encode(process.env.FASTAPI_API_SECRET);
  const expiryMinutes = Number(process.env.INTERNAL_TOKEN_EXPIRE_MINUTES ?? 60);
  return new SignJWT({ sub: userId })
    .setProtectedHeader({ alg: "HS256" })
    .setExpirationTime(`${expiryMinutes}m`)
    .sign(secret);
}
```

```python
# FastAPI 驗證端
from jose import JWTError, jwt
import os

def verify_internal_token(token: str) -> dict:
    try:
        return jwt.decode(token, os.environ["FASTAPI_API_SECRET"], algorithms=["HS256"])
    except JWTError:
        raise HTTPException(status_code=401)
```

- token 放在 server-to-server header，**不交給瀏覽器**
- `OAuth2PasswordBearer(auto_error=False)` 用於 optional auth 路由（不帶 token 時回 `None`）
- 主要 session 放 cookie，guest flow 用 opaque `guest_token` 放 `sessionStorage`

---

## 13. §D — 附加服務

### D0. FastAPI ML 依賴

- 無 GPU：`python:3.11-slim`，需要 CV 加 `apt-get install ffmpeg libgl1 libglib2.0-0`
- 有 CUDA：換 `pytorch/pytorch:2.4.0-cuda12.4-cudnn9-runtime`，`uv sync` 加 `--system`（`torch` 由 base image 提供，移出 `pyproject.toml dependencies`）
- GPU compose reservation 只加在需要 GPU 的 service；無 GPU 主機不保留 `deploy.resources`

```yaml
deploy:
  resources:
    reservations:
      devices:
        - driver: nvidia
          count: all
          capabilities: [gpu]
```

### D1. pgvector

db image 換 `pgvector/pgvector:pg16`，schema 加：
```prisma
generator client {
  previewFeatures = ["postgresqlExtensions"]
}
datasource db {
  extensions = [vector]
}
```

### D2. vLLM

用 Docker profile（`make up` 不啟動，`make up-vllm` 才啟動）：
```yaml
  vllm:
    image: vllm/vllm-openai:latest
    profiles: [vllm]
    command: --model <model-name> --dtype bfloat16 --max-model-len 8192 --port 8000
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
```

Makefile 加（注意縮排必須是 Tab）：
```makefile
up-vllm:  ## Start with vLLM
	docker compose --profile vllm up -d --build
```

啟動後 `LLM_BASE_URL` 指向 `http://vllm:8000/v1`。

---

## 14. UI：Tailwind v4 + shadcn/ui

v4 語法與 v3 不同：
```css
@import "tailwindcss";
@plugin "@tailwindcss/typography";

@theme {
  --color-brand-500: #6b7280;
  /* 其餘色票... */
}
```

`postcss.config.mjs`：
```js
export default { plugins: { "@tailwindcss/postcss": {} } };
```

不用 `@tailwind base/components/utilities`；plugin key 是 `@tailwindcss/postcss`，不是 `tailwindcss`。

---

## 15. .dockerignore

根目錄的 `.dockerignore` 對子目錄 build context 無效——§B/C 的 `fastapi-service/` 要另建一份。

根目錄：`node_modules/ .next/ .env* data/ __pycache__/ *.pyc`

`fastapi-service/`（§B/C）：`__pycache__/ *.pyc .venv/ .env*`

`.gitignore` 必備：`.env`、`data/`、`*.dump`；有 Prisma 加 `src/generated/prisma/`。

---

## 16. 生成完成檢查清單

- `docker compose config` 可正常展開
- production compose 無任何 `ports`
- `.env.example` 只含本專案需要的變數，所有 secret 是 `CHANGE_ME_...` 前綴
- 有 Prisma：`prisma generate` 與 migration 可在 container 內通過
- §B/C：FastAPI `/health` 存在；瀏覽器端無 `FASTAPI_SERVICE_URL` 直連
- 無 GPU 專案不含 `deploy.resources.devices`
- `AGENTS.md` 已刪除不適用的架構/Auth/GPU/LLM 說明

---

## 17. AGENTS.md 模板

括號內的選項擇一保留，其餘刪除。

````markdown
# <project>

## 專案概述

（一句話說明這個專案是什麼、給誰用、解決什麼問題）

## 架構

（擇一）§A：Next.js 全端，無獨立 backend，DB 由 Next.js 管（Prisma）
（擇一）§B：Next.js 主導 + FastAPI sidecar，DB 由 Next.js 管（Prisma）；FastAPI 負責（填入：推理 / CV / 長任務）
（擇一）§C：FastAPI 主導 + Next.js 薄殼，DB 由 FastAPI 管（SQLModel + Alembic）

專案名（DB user / DB name）：`<project>`
部署網址：`https://<domain>`

## Auth

（擇一）Auth-0：無登入；不建立 auth.ts / middleware.ts / login page
（擇一）Auth-1：Next.js 單一密碼（`APP_PASSWORD`），middleware.ts 保護全站
（擇一）Auth-2：Next.js 個人帳號，email/password（+ Google OAuth），layout redirect 保護
（選配）Auth-3：backend 內部授權 / guest / session token；疊加在 Auth-1/2 上，不是主要登入

## 常用指令

```bash
make up          # 啟動 production stack
make dev-up      # 啟動 dev stack（Next.js HMR）
make dev-logs    # tail 所有 container log
make db-backup   # 有 DB 時：備份 DB → backup.dump

# §A/B Prisma migration（需 dev stack 已啟動）
docker compose -f docker-compose.yml -f docker-compose.dev.yml \
  run --rm app npx prisma migrate dev --name <name>

# §A/B Prisma Studio（DB GUI）
docker compose -f docker-compose.yml -f docker-compose.dev.yml \
  run --rm -p 5555:5555 app npx prisma studio

# §C Alembic migration
docker compose run --rm fastapi alembic upgrade head
```

## 開發規則

- 本機不安裝任何 Node / Python 套件，所有指令透過 Docker 執行
- Next.js 16 + Prisma 7；Node image 使用 `node:22-bookworm-slim`
- §B/C：前端呼叫 backend 一律走 `/api/fastapi/...` proxy，不直接打 `FASTAPI_SERVICE_URL`
- 人類使用者登入放在 Next.js；backend 只處理內部 token 與資源授權（Auth-3）
- DB migration：`prisma migrate dev` 建立檔案，`entrypoint.sh` 在 production 部署時自動執行 `prisma migrate deploy`
- Prisma 7：使用 `prisma.config.ts`、`prisma-client` provider、`src/generated/prisma`、`@prisma/adapter-pg`
- Server Actions 預設；有 streaming 或外部 webhook 才改 API Route
- （專案自訂規則加在這裡）

## 安全規則

- production compose 所有 service 不暴露 ports（包含 DB）；對外只走 Cloudflare Tunnel
- dev compose 可暴露 `app:3000`，其他 service 不暴露
- .env 不 commit；secret 欄位用 `openssl rand -base64 32` 生成
- security headers 已在 next.config.ts 啟用（production only）
- （Auth-2）rate limiting 已在 signin / register route 啟用
- （專案自訂安全規則加在這裡）

## 環境

- Node 22，Python 3.11（§B/C）
- PostgreSQL 16
- GPU：（填入：不需要 / nvidia，用於 <服務名>）
````
