# Security Testing Reference (Angular + API)

**Authorized targets only:** localhost, containers, or a staging environment you own or have written permission to test. Never run these against production or third-party systems.

## Contents
1. Dependency and static checks
2. DAST with OWASP ZAP and Nuclei
3. Headers and TLS baseline
4. Auth and 2FA test matrix
5. Stress / load testing with k6
6. Angular code review grep list
7. Report template

---

## 1. Dependency and static checks

```bash
npm ci
npm audit --omit=dev --json > audit.json
npm audit signatures
npx osv-scanner --lockfile=package-lock.json
npx license-checker --production --summary      # license risk
gitleaks detect --source . --redact              # secrets in repo/history
semgrep --config p/typescript --config p/owasp-top-ten --config p/jwt .
npx eslint . --ext .ts,.html
```

Triage each advisory: direct or transitive, in the shipped bundle or dev-only, fixed version available, exploitable path in this app. Confirm with the GitHub Advisory / OSV page before upgrading.

## 2. DAST

```bash
# OWASP ZAP baseline (passive + light), staging only
docker run --rm -t -v "$PWD:/zap/wrk" ghcr.io/zaproxy/zaproxy:stable \
  zap-baseline.py -t https://staging.example.com -r zap-baseline.html

# Full active scan (staging only, with permission, off-hours)
docker run --rm -t -v "$PWD:/zap/wrk" ghcr.io/zaproxy/zaproxy:stable \
  zap-full-scan.py -t https://staging.example.com -r zap-full.html

# Known misconfigurations / CVEs
nuclei -u https://staging.example.com -severity medium,high,critical -rl 20
```

For an SPA, also feed ZAP the API OpenAPI/Swagger definition (`zap-api-scan.py -f openapi`) and run an authenticated scan with a dedicated test account (never real user data).

## 3. Headers and TLS baseline

Target response headers (tune CSP to the app; use nonces/hashes, avoid `unsafe-inline` for scripts):

```
Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-{random}'; style-src 'self' 'nonce-{random}'; img-src 'self' data:; connect-src 'self' https://api.example.com; frame-ancestors 'none'; base-uri 'self'; form-action 'self'; object-src 'none'; require-trusted-types-for 'script'
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
Cross-Origin-Opener-Policy: same-origin
Cache-Control: no-store            # on authenticated API responses
```

```bash
curl -sI https://staging.example.com | sort
testssl.sh --fast https://staging.example.com     # TLS 1.2+ only, no weak ciphers
```

Also check cookies: `HttpOnly; Secure; SameSite`, `__Host-` prefix where possible. CORS: explicit origin allow-list, no `*` with credentials.

## 4. Auth and 2FA test matrix

| Area | Test | Expected |
|---|---|---|
| Enumeration | Login/reset/register with known vs unknown user | Same message, status, and similar timing |
| Brute force | N failed logins per account and per IP | 429 or lockout, `Retry-After`, alert logged |
| Credential stuffing | Many accounts from one IP (own test accounts) | Per-IP limiter triggers |
| Session | Reuse token after logout | 401 |
| Session | Session ID changes after login | New ID (no fixation) |
| Refresh token | Reuse an already-rotated refresh token | Whole token family revoked |
| CSRF | State-changing request without token from another origin | Rejected |
| Open redirect | `returnUrl=https://evil.test` after login | Ignored / allow-list only |
| IDOR/BOLA | Access another user's resource id with your token | 403/404 |
| BFLA | Call admin endpoint as normal user | 403 |
| 2FA skip | Call a protected API with pre-auth state only | 401/403 |
| 2FA skip | Navigate straight to a protected route / replay request, omitting the 2FA call | Rejected server-side |
| 2FA replay | Submit the same valid TOTP twice in its window | Second rejected |
| 2FA brute force | 10+ wrong codes | Lockout/backoff, user notified |
| 2FA race | Parallel requests with one valid code | Only one succeeds |
| Response tamper | Edit `mfaRequired:false` / `success:true` in the response | No effect on server authorization |
| Recovery codes | Reuse a used code | Rejected; codes stored hashed |
| 2FA disable | Disable without step-up re-auth | Blocked |
| Password reset | Reuse or expired token; host header poisoning | Rejected; link uses configured base URL |
| Logout | Multi-tab | All tabs signed out |
| Step-up | Sensitive action without recent auth | Re-auth prompt enforced by the server |

Automate with Playwright (e2e flow) plus an API-level suite (REST Client / xUnit / supertest) so these become regression tests in CI.

## 5. Stress / load testing with k6 (staging only)

```javascript
// loadtest/login-journey.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  scenarios: {
    ramp: {
      executor: 'ramping-vus',
      stages: [
        { duration: '2m', target: 50 },
        { duration: '5m', target: 50 },
        { duration: '1m', target: 200 },   // spike
        { duration: '2m', target: 0 },
      ],
    },
  },
  thresholds: {
    http_req_failed: ['rate<0.01'],
    http_req_duration: ['p(95)<800', 'p(99)<1500'],
  },
};

const BASE = __ENV.BASE_URL;                // e.g. https://staging.example.com

export default function () {
  const res = http.post(`${BASE}/api/auth/login`,
    JSON.stringify({ username: `loadtest${__VU}@example.com`, password: __ENV.TEST_PASSWORD }),
    { headers: { 'Content-Type': 'application/json' } });
  check(res, { 'status ok or throttled': (r) => [200, 401, 429].includes(r.status) });
  sleep(1);
}
```

```bash
k6 run -e BASE_URL=https://staging.example.com -e TEST_PASSWORD='***' loadtest/login-journey.js
```

Scenarios to cover: login burst, 2FA verify burst, typical journey, spike, soak (30-60 min), rate-limit validation. Watch: p95/p99 latency, 5xx rate, CPU/RAM, DB connections, Redis/queue depth. Pass criteria: throttling returns 429 (not 500), the system recovers after load stops, no data corruption or duplicate side effects (idempotency on payments and writes).

Front-end performance: `npx lighthouse https://staging.example.com --view` and Lighthouse CI in the pipeline with budgets.

## 6. Angular code review grep list

```bash
grep -rnE "bypassSecurityTrust|innerHTML|outerHTML|document\.write|eval\(|new Function" src
grep -rnE "localStorage|sessionStorage" src            # tokens must not be here
grep -rnE "\[href\]|\[src\]|\[srcdoc\]" src             # check bindings with user data
grep -rnE "window\.addEventListener\('message'" src     # validate event.origin
grep -rnE "returnUrl|redirect" src                      # allow-list targets
grep -rnE "apiKey|secret|password|token" src/environments
```

## 7. Report template

```
### [SEV] Title
- Target / environment:
- Route or file:
- Description and impact:
- Steps to reproduce (minimal, non-destructive):
- Root cause:
- Fix (code/config):
- Regression test added:
- Status: open / fixed / retested (date)
```