# mostr モノレポ構築 草案

## 前提

- フロント + バックエンド1: Next.js (App Router) + TypeScript / Cloudflare Workers にデプロイ
- バックエンド2: Go / Nostr クライアント (サーバーサイド) / Docker でデプロイ
- DB: CockroachDB (PostgreSQL 互換)
- パッケージマネージャ: **npm** (npm workspaces を利用)
- データフロー:
  - Go relay が Nostr リレーから NIP イベントを受信し、CockroachDB に書き込む
  - Cloudflare Workers (Next.js) が CockroachDB から読み出し、REST / WebSocket でクライアントに配信

---

## ディレクトリ構造

```
mostr/
├── apps/
│   ├── web/                       # Next.js (App Router) + Cloudflare Workers
│   │   ├── app/
│   │   ├── public/
│   │   ├── next.config.ts
│   │   ├── open-next.config.ts    # @opennextjs/cloudflare 用
│   │   ├── wrangler.jsonc
│   │   ├── tsconfig.json
│   │   └── package.json
│   └── relay/                     # Go: Nostr client (NIP受信 → CockroachDB)
│       ├── cmd/relay/main.go
│       ├── internal/
│       │   ├── nostr/             # NIP処理、リレー接続
│       │   ├── store/             # CockroachDB アクセス層
│       │   └── config/
│       ├── Dockerfile
│       ├── go.mod
│       └── go.sum
├── packages/
│   ├── shared/                    # 共有型 (Nostr Event型, API契約 等)
│   │   ├── src/index.ts
│   │   ├── tsconfig.json
│   │   └── package.json
│   └── tsconfig/                  # 共通tsconfig (base/nextjs/node)
│       └── base.json
├── db/
│   ├── migrations/                # CockroachDB migrations (golang-migrate形式想定)
│   └── schema.sql
├── deploy/
│   └── docker-compose.yml         # ローカル開発: cockroachdb + relay
├── .editorconfig
├── .gitignore
├── .node-version
├── package.json                   # ルート (npm workspaces)
├── package-lock.json
├── turbo.json                     # 任意: ビルドキャッシュ / タスク管理
├── go.work                        # 任意: 将来 Go モジュールが増える場合
└── README.md
```

### 設計上のトレードオフ

- **npm workspaces + Turborepo**: TS 側はこの組み合わせ。Go は独立モジュールにして `go.work` でゆるく束ねる二層構成にすることで、TS 側のビルドに Go が巻き込まれない。
- **OpenNext for Cloudflare** (`@opennextjs/cloudflare`) を採用。`@cloudflare/next-on-pages` は事実上後継に置き換わっている。WebSocket は Workers の Durable Objects 側で持つのが現実的なので、将来 `apps/web` 配下に DO を追加する前提。
- **`packages/shared`**: Nostr Event の型は Go/TS 双方で必要。TS はここで定義し、Go は `internal/nostr/types.go` で別実装にするのが摩擦が少ない (自動生成は最初は不要)。
- **npm workspaces のローカル参照**: `workspace:*` 記法は npm では未対応のため、依存指定は `"*"` を使う。

---

## 構築コマンド (順番に実行)

### 1. ルート初期化

```bash
cd /home/maretol/project/mostr

# Node 固定 (任意)
echo "22" > .node-version

# ルート package.json を作成
npm init -y

# 依存追加 (ルートに置く開発ツール)
npm install -D turbo typescript @types/node
```

`package.json` を以下のように編集:

```json
{
  "name": "mostr",
  "private": true,
  "workspaces": [
    "apps/*",
    "packages/*"
  ],
  "scripts": {
    "dev": "turbo run dev",
    "build": "turbo run build",
    "lint": "turbo run lint",
    "typecheck": "turbo run typecheck"
  }
}
```

### 2. Next.js (apps/web) — Cloudflare Workers

```bash
mkdir -p apps
cd apps
npx create-next-app@latest web --ts --app --tailwind --eslint --no-src-dir --import-alias "@/*"
cd web
npm install -D @opennextjs/cloudflare wrangler
# Cloudflare 用初期化 (wrangler.jsonc / open-next.config.ts を生成)
npx @opennextjs/cloudflare@latest init
cd ../..
```

### 3. 共通パッケージ

```bash
mkdir -p packages/shared/src packages/tsconfig

# packages/shared
cd packages/shared
npm init -y
# package.json の name を "@mostr/shared" に、main/types を src/index.ts に変更
: > src/index.ts
cd ../..

# 共通 tsconfig
cat > packages/tsconfig/base.json <<'EOF'
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "isolatedModules": true,
    "resolveJsonModule": true
  }
}
EOF
```

`apps/web/package.json` の `dependencies` に `"@mostr/shared": "*"` を追加 → ルートで `npm install` 実行。

### 4. Go relay (apps/relay)

```bash
mkdir -p apps/relay/cmd/relay apps/relay/internal/{nostr,store,config}
cd apps/relay
go mod init github.com/maretol/mostr/apps/relay
# Nostr クライアントライブラリ (代表例)
go get github.com/nbd-wtf/go-nostr
# CockroachDB は pgx で接続
go get github.com/jackc/pgx/v5
cat > cmd/relay/main.go <<'EOF'
package main

func main() {}
EOF
cd ../..

# go.work でモノレポ統合 (Go モジュールが増えても拡張容易)
go work init ./apps/relay
```

`apps/relay/Dockerfile` (multi-stage):

```dockerfile
FROM golang:1.23-alpine AS build
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /out/relay ./cmd/relay

FROM gcr.io/distroless/static-debian12
COPY --from=build /out/relay /relay
ENTRYPOINT ["/relay"]
```

### 5. ローカル開発用 docker-compose

```bash
mkdir -p deploy db/migrations
cat > deploy/docker-compose.yml <<'EOF'
services:
  cockroach:
    image: cockroachdb/cockroach:latest-v24.3
    command: start-single-node --insecure
    ports: ["26257:26257", "8080:8080"]
    volumes: ["cockroach-data:/cockroach/cockroach-data"]
  relay:
    build:
      context: ../apps/relay
    depends_on: [cockroach]
    environment:
      DATABASE_URL: postgresql://root@cockroach:26257/mostr?sslmode=disable
volumes:
  cockroach-data:
EOF
```

### 6. ルート .gitignore と turbo.json

```bash
cat > .gitignore <<'EOF'
node_modules/
.next/
.open-next/
.wrangler/
dist/
out/
*.log
.env
.env.local
.DS_Store
.turbo/
apps/relay/relay
EOF

cat > turbo.json <<'EOF'
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build":     { "dependsOn": ["^build"], "outputs": [".next/**", ".open-next/**", "dist/**"] },
    "dev":       { "cache": false, "persistent": true },
    "lint":      {},
    "typecheck": { "dependsOn": ["^build"] }
  }
}
EOF
```

---

## 進め方

推奨順:

1. ルート + Next.js (`apps/web`) のセットアップ
2. Go relay (`apps/relay`) のスケルトン作成
3. docker-compose で CockroachDB と疎通確認
4. 共有型 (`packages/shared`) と DB スキーマ (`db/`) の整備

## 未決事項 / 今後の検討

- Cloudflare Workers から CockroachDB への接続方法 (Hyperdrive 経由 / 直接 / 自前プロキシ)
- WebSocket 実装場所 (Durable Objects か、別途 Go 側か)
- Nostr Event 型を TS / Go で共有する手段 (手書き維持 or スキーマ駆動の自動生成)
- マイグレーションツールの選定 (golang-migrate / dbmate / atlas など)
- CI / CD (GitHub Actions の構成、Cloudflare デプロイ、Docker イメージのレジストリ)
