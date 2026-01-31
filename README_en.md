# Moltworker × Cloudflare Zero Trust — Minimal Reproducible Setup (EN)

This repository documents the **minimal, reproducible Cloudflare-based setup**
for running **moltworker** with:

- Browser access (Google Login via Cloudflare Access)
- CLI / service-to-service access (Service Token)
- Persistent storage (Cloudflare R2)

Target: **re-install in 30–45 minutes** without guesswork.

---

## Architecture

Browser / CLI  
→ Cloudflare Access (Google IdP)  
→ `agent.<your-domain>`  
→ Cloudflare Worker (moltworker)  
→ R2 (persistent state)

---

## Requirements

- Cloudflare account (Owner access)
- Custom domain (e.g. `example.com`)
- Google account (for OAuth IdP)
- Node.js + npm
- Cloudflare Wrangler CLI

```bash
npm install -g wrangler
wrangler login
```

---

## 1) Enable Cloudflare Zero Trust

Cloudflare Dashboard → **Cloudflare One** → Get started

- Set a **Team name**
- Note the generated team domain:

```
<team-name>.cloudflareaccess.com
```

This becomes `CF_ACCESS_TEAM_DOMAIN`.

---

## 2) Billing plan

Use **Zero Trust Standard**:

- Seats: **1**
- Cost: **$7 / month**

Free plan frequently causes login / seat issues for browser-based use.

---

## 3) Google Login (Identity Provider)

### 3.1 Google Cloud

1. Create a new Google Cloud project
2. OAuth consent screen
   - Type: **External**
3. Create OAuth client
   - Type: **Web**
   - Authorized redirect URI:

```
https://<TEAM_DOMAIN>/cdn-cgi/access/callback
```

Save:
- Client ID
- Client Secret

### 3.2 Cloudflare

Cloudflare One → **Integrations → Identity providers → Add Google**

- Name: Google
- App ID: Google Client ID
- Client secret: Google Client Secret
- PKCE: OFF

---

## 4) Cloudflare Access application

Cloudflare One → **Access → Applications → Add application**

- Type: Self-hosted
- Name: `moltbot`
- Public hostname:

```
agent.example.com
```

### Policies

- ALLOW (Google IdP / your account)
- SERVICE AUTH (Service Token) for CLI automation (recommended)

---

## 5) Deploy moltworker

```bash
git clone https://github.com/cloudflare/moltworker
cd moltworker
npm install
wrangler deploy
```

---

## 6) Worker route (critical)

Workers & Pages → your worker → **Settings → Domains & Routes**

Add route:

```
agent.example.com/*
```

---

## 7) Service Token (CLI / automation)

Cloudflare One → **Service credentials** → Create service token

- Name: `moltbot-cli`
- Save:
  - Client ID
  - Client Secret

Attach to Access Application policy:
- Action: **SERVICE AUTH**
- Selector: Service Token
- Value: `moltbot-cli`

---

## 8) R2 Storage (persistence)

### 8.1 Create bucket

R2 → Create bucket

```
moltbot-data
```

### 8.2 Create R2 API Token (S3-compatible)

R2 → API Tokens → create an API token with:

- Permissions: **Object Read & Write**
- Scope: only bucket `moltbot-data`
- TTL: Forever (or long)

Save:
- `R2_ACCESS_KEY_ID`
- `R2_SECRET_ACCESS_KEY`
- `CF_ACCOUNT_ID` (your Cloudflare account ID)

---

## 9) Set Worker secrets

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

## 10) Restart gateway

Open:

```
https://agent.example.com/_admin/
```

Click **Restart Gateway**.

---

## 11) Verify

### Browser

```
https://agent.example.com
```

→ Google login → UI loads

### CLI (Service Token)

```bash
curl -i https://agent.example.com \
  -H "CF-Access-Client-Id: <CLIENT_ID>" \
  -H "CF-Access-Client-Secret: <CLIENT_SECRET>"
```

Expect **200 OK**.

---

## Notes

- Disable/remove One-time PIN once Google login is working (recommended)
- Service Token is for automation; browser uses user login
- R2 is required for persistence
