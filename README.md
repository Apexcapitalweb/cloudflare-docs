**Cloudflare Access policies** — how they work and how they apply to Apex Capital.

### What Access does
Cloudflare Access (Zero Trust) sits in front of an application (hostname, path, or Worker) and decides **who** may reach it before traffic hits your origin or Worker.

Policies are evaluated per Access **application** (e.g. `apexcapitalweb.com`, `api.apexcapitalweb.com`).

---

### Policy building blocks

Every policy has:

| Piece | Meaning |
|--------|---------|
| **Action** | What happens on match |
| **Rule types** | How criteria combine |
| **Selectors + values** | What is checked (email, country, token, …) |

#### Actions

| Action | Effect |
|--------|--------|
| **Allow** | Authenticated users who match may enter (IdP / OTP login) |
| **Block** | Explicit deny |
| **Bypass** | No Access enforcement (public). Use sparingly |
| **Service Auth** | Non-interactive auth (service tokens, mTLS) — no IdP login UI |

#### Rule types (logic)

| Type | Logic | Role |
|------|--------|------|
| **Include** | OR | Who is in the candidate pool (required) |
| **Require** | AND | Extra conditions everyone must satisfy |
| **Exclude** | NOT | Hard deny even if Include/Require match |

Flow: **Include** (eligible) → **Require** (must also pass) → **Exclude** (never allowed).

---

### Example policies

**Allow one operator (matches your README lockout checklist)**

| Action | Rule type | Selector | Value |
|--------|-----------|----------|--------|
| Allow | Include | Email | `xiaunknown116@gmail.com` |

**Allow domain, exclude two users**

| Action | Rule type | Selector | Value |
|--------|-----------|----------|--------|
| Allow | Include | Emails ending in | `@team.com` |
| | Exclude | Email | `user-1@team.com`, `user-2@team.com` |

**Country + domain**

| Action | Rule type | Selector | Value |
|--------|-----------|----------|--------|
| Allow | Include | Country | Portugal |
| | Require | Emails ending in | `@team.com` |

---

### Service tokens (machines / health / CI)

For scripts, monitors, or `curl` without a browser login:

1. **Zero Trust → Access controls → Service credentials → Service Tokens** → create token  
2. Copy **Client ID** and **Client Secret** (secret shown once)  
3. Policy on the app:

| Action | Rule type | Selector | Value |
|--------|-----------|----------|--------|
| **Service Auth** | Include | Service Token | *your token* |

4. Call with headers:

```bash
curl -H "CF-Access-Client-Id: <CLIENT_ID>" \
     -H "CF-Access-Client-Secret: <CLIENT_SECRET>" \
     https://apexcapitalweb.com/api/health
```

Use **Service Auth**, not Allow, or Access will still send browsers to the login page.

---

### Recommended layout for Apex

| Surface | Access approach |
|---------|------------------|
| **Public marketing site** | Often **no** Access, or Bypass for everyone |
| **Investor / app UI** | Allow specific emails or `@yourdomain` |
| **Admin control plane** | Allow only admin emails + short session; later MFA / dual-control |
| **`GET /api/health`** | Separate app path **or** Service Auth token for probes; avoid locking out monitors |
| **`POST /api/admin/*`** | Strict Allow (admins only); Worker still enforces `x-admin-token` |

**Path-based apps** (optional):  
- App A: `apexcapitalweb.com/api/health` → Bypass or Service Auth  
- App B: `apexcapitalweb.com` → Allow operator email  

Policy order matters: more specific applications/paths should be defined so health is not forced through the same login as admin UI.

---

### Dashboard path

1. [Cloudflare Dashboard](https://dash.cloudflare.com) → **Zero Trust**  
2. **Access controls → Applications** → create/select app for `apexcapitalweb.com`  
3. Add policies (Include email, etc.)  
4. Save → test in **incognito** (clears Access cookies)

IdP options: One-time PIN (email), Google, GitHub, Entra ID, generic OIDC, etc. OTP alone is enough for a single-operator sandbox.

---

### Common failure modes (your checklist)

| Symptom | Likely cause |
|---------|----------------|
| Infinite Access login | Email not in **Include**, or in **Exclude** |
| Works in one browser only | Old Access cookie → try incognito |
| Service token → 302 to login | Policy is **Allow** instead of **Service Auth**, or wrong Client ID/secret |
| Health checks fail | Health URL covered by strict Allow with no token/Bypass |
| Zone still **pending** | Nameservers not delegated — Access never runs until the zone is active |

---

### Relation to the Worker

Access runs **at the edge in front of** the Worker. Even after Access allows a request:

- CORS, body limits, and timing-safe `ADMIN_TOKEN` in `apex-sandbox-api` still apply  
- Access identity can be read in a Worker via Access JWT / `ctx.access` when configured  

Defense in depth: Access = who reaches the host; Worker = what the API allows.

---

### Practical default for you right now

**Application:** `apexcapitalweb.com` (and API host if separate)

**Policy 1 — Operators**  
- Action: **Allow**  
- Include: Email = `xiaunknown116@gmail.com`  

**Policy 2 — Health probes (optional)**  
- Separate application on `/api/health` **or** Service Auth with a dedicated token  

Then re-test in incognito after save.

Official docs: [Access policies](https://developers.cloudflare.com/cloudflare-one/access-controls/policies/), [Service tokens](https://developers.cloudflare.com/cloudflare-one/access-controls/service-credentials/service-tokens/), [Common policies](https://developers.cloudflare.com/cloudflare-one/access-controls/policies/common-policies/).

I can draft exact policy tables for “public site + locked admin + tokenized health” next if you want that written as a checklist for the dashboard.# Cloudflare Developer Documentation

Welcome to the open-source repository for all [Cloudflare's Documentation](https://developers.cloudflare.com/).

To learn how to contribute, visit the [Cloudflare Style Guide](https://developers.cloudflare.com/style-guide/contributions/).

## License and Legal Notices

Except as otherwise noted, Cloudflare and any contributors grant you a license to the Cloudflare Developer Documentation and other content in this repository under the [Creative Commons Attribution 4.0 International Public License](https://creativecommons.org/licenses/by/4.0/legalcode), see the [LICENSE file](https://github.com/cloudflare/cloudflare-docs/blob/production/LICENSE), and grant you a license to any code in the repository under the [MIT License](https://opensource.org/licenses/MIT), see the [LICENSE-CODE file](https://github.com/cloudflare/cloudflare-docs/blob/production/LICENSE-CODE).

Cloudflare products and services referenced in the documentation may be either trademarks or registered trademarks of Cloudflare in the United States and/or other countries. The licenses for this project do not grant you rights to use any Cloudflare names, logos, or trademarks. Cloudflare's general trademark guidelines can be found at [https://www.cloudflare.com/trademark/](https://www.cloudflare.com/trademark/).
Cloudflare and any contributors reserve all other rights, whether under their respective copyrights, patents, or trademarks, whether by implication, estoppel, or otherwise.

We may use AI tools to help us review technical documentation, pull requests and other issues submitted to our public GitHub page in order to identify and correct mistakes and other inconsistencies in our developer documentation. Please refrain from sharing any personal information in your submissions.
