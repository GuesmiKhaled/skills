---
id: "B-04"
titre: "Threads — Virtual Threads, pinning & sécurité de la concurrence"
severite: "🔴 BLOCAGE (InterruptedException avalée / wait sans while) — 🟡 ALERTE"
applicable_si: [backend]
declencheurs:
  - "Code I/O bloquant, synchronized, ThreadPoolExecutor, ThreadLocal dans {{MODULE_CIBLE}}"
  - "Tout bloc try/catch sur InterruptedException"
  - "Tout appel à Object.wait(), Thread.sleep(), await()"
  - "Toute variable partagée entre threads sans synchronisation explicite"
  - "SonarQube S2142 / S2274"
version: "1.0"
---

## Risque
`InterruptedException` avalée → thread zombie au graceful shutdown → traitement en double à la reprise.
`synchronized` sur I/O avec Virtual Threads → pinning du carrier thread → annule le bénéfice des VT (fiche B-07).
`Object.wait()` dans un `if` → réveil spurieux → bug de concurrence silencieux.

## Règles

**Virtual Threads (pinning) :**
- `synchronized` sur un bloc I/O (BDD, réseau, file) → `ReentrantLock` (non-pinned en Java 21).
- `ThreadPoolExecutor` de taille fixe pour l'I/O → exécuteur VT non borné.
- `ThreadLocal` sans `remove()` → passer à `ScopedValue` ou documenter le cycle de vie.

**Sécurité de la concurrence (Sonar S2142/S2274) :**
- `InterruptedException` → **toujours** restaurer le flag : `Thread.currentThread().interrupt()` + return propre. Ou propager via `throws InterruptedException`.
- `Object.wait()` → **toujours** dans un `while (condition)`, jamais dans un `if`.
- Variable partagée mutable → `AtomicXxx`, `ConcurrentHashMap`, ou `volatile`.

## Anti-pattern
```java
// ❌ BLOCAGE S2142 — exception avalée, thread zombie
try {
    Thread.sleep(retryDelay);
} catch (InterruptedException e) {
    log.warn("Interrupted"); // flag perdu → thread ne s'arrête jamais proprement
}

// ❌ BLOCAGE S2274 — réveil spurieux possible
synchronized (lock) {
    if (!ready) {
        lock.wait(); // réveil spurieux → traitement prématuré
    }
}

// ❌ ALERTE — pinning du virtual thread
synchronized (this) {
    // appel I/O — bloque le carrier thread
}
```

## Pattern conforme
```java
// ✅ InterruptedException — flag restauré
try {
    Thread.sleep(retryDelay);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // flag restauré
    log.warn("Thread interrupted in {{MODULE_CIBLE}}, stopping processing");
    return; // sortir proprement → graceful shutdown respecté
}

// ✅ wait() dans while — protège contre les réveils spurieux
synchronized (lock) {
    while (!ready) {
        lock.wait(); // boucle obligatoire
    }
    process();
}

// ✅ ReentrantLock — non-pinned sur I/O avec Virtual Threads
private final ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    // appel I/O non-bloquant pour le carrier
} finally {
    lock.unlock();
}

// ✅ Variable partagée — AtomicLong
private final AtomicLong counter = new AtomicLong(0);
counter.incrementAndGet(); // atomique, sans synchronized
```

## Checklist CI/CD
- [ ] `grep -r "catch (InterruptedException" src/ | grep -v "interrupt()"` → 0
- [ ] `grep -r "\.wait()" src/` → vérifier que chaque occurrence est dans un `while`
- [ ] Sonar S2142 et S2274 → "Resolved" ou "Safe", jamais "Won't Fix"
- [ ] `grep -r "synchronized" src/ | grep -i "{{IO_KEYWORDS}}"` → remplacer par ReentrantLock
- [ ] Test : thread interrompu → `interrupt()` restauré → scheduler s'arrête proprement
