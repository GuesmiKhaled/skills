---
id: "S-03"
titre: "Sécurité des callbacks — HMAC + anti-rejeu"
severite: "🔴 BLOCAGE — callback à effet sans signature ou sans anti-rejeu"
applicable_si: [backend]
declencheurs:
  - "Création d'un endpoint webhook/callback/notification dans {{MODULE_CIBLE}}"
  - "Tout endpoint d'action avec effet (création, débit, modification d'état)"
  - "Tout callback externe reçu d'un système tiers"
version: "1.0"
---

## Risque
Sans HMAC vérifié : n'importe qui atteignant l'endpoint peut forger des callbacks → effet non autorisé.
Sans anti-rejeu : une requête valide capturée et rejouée N fois = N effets indésirables. L'authentification dit *qui parle*, pas *quand* ni *une seule fois*.

## Règles

**HMAC (authentification du callback) :**
- Tout callback à effet passe par `{{SIGNATURE_FILTER}}` avant le handler.
- Algorithme : `HMAC-SHA256(clé, timestamp + "." + body)`.
- Comparaison en **temps constant** — jamais `String.equals()` (timing attack).
- Clé HMAC dans un gestionnaire de secrets ({{SECRET_STORE}}) — jamais en dur.

**Anti-rejeu (unicité dans le temps) :**
- En-têtes requis : `X-Timestamp` (UTC ISO-8601), `X-Nonce` (UUID), `{{IDEMPOTENCY_HEADER}}`.
- `X-Timestamp` vérifié à ± 5 min contre l'horloge serveur — jamais l'horloge client.
- `X-Nonce` → `store.tryStore(nonce, TTL=10min)` — si déjà présent → `409 Replay detected`.
- Nonce stocké en cache distribué ({{CACHE_STORE}}) — jamais en mémoire seule.

## Anti-pattern
```java
// ❌ BLOCAGE — callback sans signature
@PostMapping("/{{CALLBACK_ENDPOINT}}")
public void handle(@RequestBody Event event) {
    service.process(event); // n'importe qui peut POSTer
}

// ❌ BLOCAGE — comparaison sensible au timing
if (received.equals(computed)) { ... }
```

## Pattern conforme
```java
// ✅ Filtre de signature AVANT le handler
public class SignatureFilter extends OncePerRequestFilter {
    @Override protected void doFilterInternal(HttpServletRequest req, ...) {
        String timestamp = req.getHeader("X-Timestamp");
        String nonce     = req.getHeader("X-Nonce");

        // 1. Fenêtre temporelle ± 5 min
        if (isExpired(timestamp)) { resp.sendError(401); return; }

        // 2. Nonce unique (TTL 10 min)
        if (!nonceStore.tryStore(nonce)) { resp.sendError(409, "Replay"); return; }

        // 3. HMAC en temps constant
        String expected = hmacSha256(secret, timestamp + "." + body);
        if (!MessageDigest.isEqual(received.getBytes(), expected.getBytes())) {
            resp.sendError(401); return;
        }
        chain.doFilter(req, resp);
    }
}
```

## Checklist CI/CD
- [ ] Tout `POST /{{CALLBACK_ENDPOINT}}` passe par `{{SIGNATURE_FILTER}}`
- [ ] Clé HMAC dans {{SECRET_STORE}} — `grep -r "hmacSecret\s*=" src/` → 0 valeur littérale
- [ ] Test : signature valide→200 ; fausse→401 ; timestamp expiré→401 ; nonce rejoué→409 ; double delivery→1 effet
