# Moltworker × Cloudflare Zero Trust — Minimal Reproducible Setup

[English](#english) | [日本語](#日本語)

---

## English

This repository documents the **minimal, reproducible Cloudflare-based setup**
for running **moltworker** with:

- Browser access (Google Login via Cloudflare Access)
- CLI / service-to-service access (Service Token)
- Persistent storage (Cloudflare R2)

**Goal:** Re-install in **30–45 minutes** with no guesswork.

### Architecture

```
Browser / CLI
→ Cloudflare Access (Google IdP)
→ agent.<your-domain>
→ Cloudflare Worker (moltworker)
→ R2 (persistent state)
```

### Requirements

- Cloudflare account (Owner access)
- Custom domain (e.g. `example.com`)
- Google account (for OAuth IdP)
- Node.js + npm
- Cloudflare Wrangler CLI

```bash
npm install -g wrangler
wrangler login
```

### 1) Enable Cloudflare Zero Trust

Cloudflare Dashboard → **Cloudflare One** → Get started

- Set a **Team name**
- Note the generated team domain:

```
<team-name>.cloudflareaccess.com
```

This becomes `CF_ACCESS_TEAM_DOMAIN`.

### 2) Billing Plan

Use **Zero Trust Standard**:
- Seats: **1**
- Cost: **$7 / month**

### 3) Google Login (Identity Provider)

**Google Cloud**
1. Create a new Google Cloud project
2. OAuth consent screen → **External**
3. Create OAuth client (Web)
4. Authorized redirect URI:

```
https://<TEAM_DOMAIN>/cdn-cgi/access/callback
```

Save **Client ID / Client Secret**.

**Cloudflare**

Cloudflare One → Integrations → Identity providers → Add Google
- App ID: Google Client ID
- Client secret: Google Client Secret
- PKCE: OFF

### 4) Cloudflare Access Application

Cloudflare One → Access → Applications → Add application
- Type: Self-hosted
- Name: `moltbot`
- Public hostname: `agent.example.com`

Policies:
- ALLOW (Google IdP)
- SERVICE AUTH (Service Token) for CLI automation

### 5) Deploy moltworker

```bash
git clone https://github.com/cloudflare/moltworker
cd moltworker
npm install
wrangler deploy
```

### 6) Worker Route

Workers & Pages → Worker → Settings → Domains & Routes

```
agent.example.com/*
```

### 7) Service Token (CLI)

Cloudflare One → Service credentials → Create service token
- Name: `moltbot-cli`
- Save Client ID / Client Secret
- Attach to Access policy with **SERVICE AUTH**

### 8) R2 Storage

Create bucket:
```
moltbot-data
```

Create R2 API token (S3-compatible):
- Permissions: Object Read & Write
- Scope: `moltbot-data`
- TTL: Forever

Save:
- `R2_ACCESS_KEY_ID`
- `R2_SECRET_ACCESS_KEY`
- `CF_ACCOUNT_ID`

### 9) Set Worker Secrets

```bash
wrangler secret put R2_ACCESS_KEY_ID
wrangler secret put R2_SECRET_ACCESS_KEY
wrangler secret put CF_ACCOUNT_ID

wrangler secret put CF_ACCESS_TEAM_DOMAIN
wrangler secret put CF_ACCESS_AUD
wrangler secret put CF_ACCESS_CLIENT_ID
wrangler secret put CF_ACCESS_CLIENT_SECRET
```

### 10) Restart Gateway

```
https://agent.example.com/_admin/
```

Click **Restart Gateway**.

### 11) Verify

Browser:
```
https://agent.example.com
```

CLI:
```bash
curl -i https://agent.example.com \
  -H "CF-Access-Client-Id: <CLIENT_ID>" \
  -H "CF-Access-Client-Secret: <CLIENT_SECRET>"
```

---

## 日本語

このリポジトリは **Cloudflare 上で moltworker を動かすための最小・再現可能な手順** をまとめたものです。

**目的**
- ブラウザから利用（Cloudflare Access + Google ログイン）
- CLI / 自動化（Service Token）
- 永続化（Cloudflare R2）
- **30〜45分で再構築**できること

### 構成

```
Browser / CLI
→ Cloudflare Access（Google IdP）
→ agent.<あなたのドメイン>
→ Cloudflare Worker（moltworker）
→ R2（状態永続化）
```

### 必要なもの

- Cloudflare アカウント（Owner 権限）
- 独自ドメイン
- Google アカウント
- Node.js / npm
- Wrangler CLI

```bash
npm install -g wrangler
wrangler login
```

### 1) Cloudflare Zero Trust 有効化

Cloudflare Dashboard → **Cloudflare One** → Get started

- Team name 設定
- 自動生成される Team domain を控える：

```
<team-name>.cloudflareaccess.com
```

これが `CF_ACCESS_TEAM_DOMAIN` になります。

### 2) 課金プラン

**Zero Trust Standard**
- Seats: **1**
- 料金: **$7 / 月**

### 3) Google ログイン（IdP）

**Google Cloud**
1. 新規プロジェクト作成
2. OAuth 同意画面 → External
3. OAuth クライアント作成（Web）
4. リダイレクト URI：

```
https://<TEAM_DOMAIN>/cdn-cgi/access/callback
```

**Client ID / Client Secret** を保存。

**Cloudflare**

Cloudflare One → Integrations → Identity providers → Add Google
- App ID: Google Client ID
- Client secret: Google Client Secret
- PKCE: OFF

### 4) Access アプリ

Cloudflare One → Access → Applications → Add application
- Type: Self-hosted
- Name: `moltbot`
- Hostname: `agent.example.com`

ポリシー:
- ALLOW（Google IdP）
- SERVICE AUTH（Service Token）CLI 自動化用

### 5) moltworker デプロイ

```bash
git clone https://github.com/cloudflare/moltworker
cd moltworker
npm install
wrangler deploy
```

### 6) Route 設定

Workers & Pages → Worker → Settings → Domains & Routes

```
agent.example.com/*
```

### 7) Service Token

Cloudflare One → Service credentials → Create service token
- Name: `moltbot-cli`
- Client ID / Client Secret を保存
- Access policy に **SERVICE AUTH** として追加

### 8) R2 ストレージ

Bucket 作成:
```
moltbot-data
```

R2 API Token 作成（S3互換）:
- Permissions: Object Read & Write
- Scope: `moltbot-data`
- TTL: Forever

保存:
- `R2_ACCESS_KEY_ID`
- `R2_SECRET_ACCESS_KEY`
- `CF_ACCOUNT_ID`

### 9) Worker Secrets 設定

```bash
wrangler secret put R2_ACCESS_KEY_ID
wrangler secret put R2_SECRET_ACCESS_KEY
wrangler secret put CF_ACCOUNT_ID

wrangler secret put CF_ACCESS_TEAM_DOMAIN
wrangler secret put CF_ACCESS_AUD
wrangler secret put CF_ACCESS_CLIENT_ID
wrangler secret put CF_ACCESS_CLIENT_SECRET
```

### 10) Gateway 再起動

```
https://agent.example.com/_admin/
```

**Restart Gateway** をクリック。

### 11) 動作確認

ブラウザ:
```
https://agent.example.com
```

CLI:
```bash
curl -i https://agent.example.com \
  -H "CF-Access-Client-Id: <CLIENT_ID>" \
  -H "CF-Access-Client-Secret: <CLIENT_SECRET>"
```
