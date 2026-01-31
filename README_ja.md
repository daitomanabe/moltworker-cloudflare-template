# Moltworker × Cloudflare Zero Trust — 最小再現セットアップ（JA）

このリポジトリは **Cloudflare 上で moltworker を動かすための最小・再現可能な手順** をまとめたものです。

目的：
- **ブラウザから使える**（Cloudflare Access + Google ログイン）
- **CLI / 自動化もできる**（Service Token）
- **永続化できる**（R2）
- 数か月後でも **30〜45分で再構築**できる

---

## 構成（ゴール）

Browser / CLI  
→ Cloudflare Access（Google IdP）  
→ `agent.<あなたのドメイン>`  
→ Cloudflare Worker（moltworker）  
→ R2（状態を永続化）

---

## 必要なもの

- Cloudflare アカウント（Owner 権限）
- 独自ドメイン（例：`example.com`）
- Google アカウント（IdP 用）
- Node.js + npm
- Cloudflare Wrangler CLI

```bash
npm install -g wrangler
wrangler login
```

---

## 1) Cloudflare Zero Trust を有効化

Cloudflare Dashboard → **Cloudflare One** → Get started

- Team name を設定
- 自動生成される Team domain を控える：

```
<team-name>.cloudflareaccess.com
```

これが `CF_ACCESS_TEAM_DOMAIN` です。

---

## 2) 課金プラン

**Zero Trust Standard** を使用：

- Seats：**1**
- 料金：**$7 / month**

※ Free のままだとブラウザログインや seat 周りで詰みやすいです。

---

## 3) Google ログイン（Identity Provider）

### 3.1 Google Cloud 側

1. 新しい Google Cloud プロジェクトを作成
2. OAuth 同意画面
   - Type：**External**
3. OAuth クライアント作成
   - Type：**Web**
   - Authorized redirect URI：

```
https://<TEAM_DOMAIN>/cdn-cgi/access/callback
```

控えるもの：
- Client ID
- Client Secret

### 3.2 Cloudflare 側

Cloudflare One → **Integrations → Identity providers → Add Google**

- Name：Google
- App ID：Google Client ID
- Client secret：Google Client Secret
- PKCE：OFF

---

## 4) Cloudflare Access アプリ（Self-hosted）

Cloudflare One → **Access → Applications → Add application**

- Type：Self-hosted
- Name：`moltbot`
- Public hostname：

```
agent.example.com
```

### ポリシー（推奨）

- ALLOW：Google ログイン（自分）
- SERVICE AUTH：Service Token（CLI / 自動化）

---

## 5) moltworker をデプロイ

```bash
git clone https://github.com/cloudflare/moltworker
cd moltworker
npm install
wrangler deploy
```

---

## 6) Worker Route（最重要）

Workers & Pages → 対象 worker → **Settings → Domains & Routes**

Route を追加：

```
agent.example.com/*
```

---

## 7) Service Token（CLI / 自動化）

Cloudflare One → **Service credentials** → Create service token

- Name：`moltbot-cli`
- 控える：
  - Client ID
  - Client Secret

Access アプリのポリシーに追加：
- Action：**SERVICE AUTH**
- Selector：Service Token
- Value：`moltbot-cli`

---

## 8) R2（永続化）

### 8.1 バケット作成

R2 → Create bucket

```
moltbot-data
```

### 8.2 R2 API Token（S3互換）を作成

R2 → API Tokens → 作成（以下）

- Permissions：**Object Read & Write**
- Scope：bucket `moltbot-data` のみ
- TTL：Forever（または長期）

控えるもの：
- `R2_ACCESS_KEY_ID`
- `R2_SECRET_ACCESS_KEY`
- `CF_ACCOUNT_ID`（Cloudflare の Account ID）

---

## 9) Worker secrets を投入

```bash
wrangler secret put R2_ACCESS_KEY_ID
wrangler secret put R2_SECRET_ACCESS_KEY
wrangler secret put CF_ACCOUNT_ID

wrangler secret put CF_ACCESS_TEAM_DOMAIN
wrangler secret put CF_ACCESS_AUD
wrangler secret put CF_ACCESS_CLIENT_ID
wrangler secret put CF_ACCESS_CLIENT_SECRET
```

---

## 10) Gateway 再起動

```
https://agent.example.com/_admin/
```

で **Restart Gateway** を押す。

---

## 11) 動作確認

### ブラウザ

```
https://agent.example.com
```

→ Google ログイン → UI 表示

### CLI（Service Token）

```bash
curl -i https://agent.example.com \
  -H "CF-Access-Client-Id: <CLIENT_ID>" \
  -H "CF-Access-Client-Secret: <CLIENT_SECRET>"
```

`200 OK` なら成功。

---

## 補足

- Google が動いたら One-time PIN は OFF/削除推奨
- Service Token は自動化用。ブラウザはユーザーログイン
- R2 は永続化に必須
