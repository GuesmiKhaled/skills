---
id: "B-06"
titre: "Saga — transactions distribuées sans 2PC"
severite: "🔴 BLOCAGE (2PC/XA inter-services)"
applicable_si: [backend]
declencheurs:
  - "Tout flux multi-services nécessitant une cohérence transactionnelle dans {{MODULE_CIBLE}}"
  - "Tout @Transactional couvrant plusieurs services ou plusieurs BDD"
  - "Tout état métier sans machine d'état explicite"
version: "1.0"
---

## Risque
2PC/XA = verrou distribué : si un participant tombe après le vote PREPARE → toute la transaction reste gelée jusqu'à la récupération manuelle.
En production : un état gelé indéfiniment = client impacté sans confirmation → P1.

## Règles

- Jamais de 2PC/XA entre microservices (jamais de `@Transactional` couvrant plusieurs BDD).
- Tout flux multi-services → **Saga chorégraphiée** (events) ou **orchestrée** (Saga orchestrator).
- Chaque étape → état explicite : `{{STATE_INITIAL}}` → `{{STATE_INTERMEDIATE}}` → `{{STATE_FINAL}}` (ou `{{STATE_FAILED}}` + compensation).
- Compensation → idempotente (fiche B-02) et tracée (log + événement).
- État intermédiaire → toujours réconcilié (scheduler de réconciliation).

## Anti-pattern
```java
// ❌ BLOCAGE — @Transactional couvrant plusieurs services/BDD
@Transactional
public void processFlow({{COMMAND}} cmd) {
    repository.save(entity);                    // BDD A
    {{EXTERNAL_SERVICE_A}}.create(entity);      // appel HTTP → BDD B
    {{EXTERNAL_SERVICE_B}}.notify(entity);      // appel HTTP → BDD C
    // Si notify() échoue → rollback impossible sur B et C
}
```

## Pattern conforme
```java
// ✅ Saga chorégraphiée par événements

// Étape 1 — Service A
@Transactional
public {{ENTITY}} step1({{COMMAND}} cmd) {
    {{ENTITY}} entity = new {{ENTITY}}(cmd, {{STATE_INITIAL}});
    repository.save(entity);
    messagingClient.send("{{EVENT_STEP1}}",
        new {{EVENT_STEP1_TYPE}}(entity.id(), cmd.idempotencyKey()));
    return entity;
}

// Étape 2 — Service B (consumer at-least-once + idempotence)
void onStep1({{EVENT_STEP1_TYPE}} event, Acknowledgment ack) {
    if (!repositoryB.existsBy(event.entityId())) {
        repositoryB.save({{ENTITY_B}}.from(event));
        messagingClient.send("{{EVENT_STEP2}}", ...);
    }
    ack.acknowledge();
}

// ✅ Compensation — état FAILED, pas un rollback
void onFailed({{EVENT_FAILED_TYPE}} event, Acknowledgment ack) {
    repositoryB.findBy(event.entityId())
        .ifPresent(e -> repositoryB.save(e.withStatus({{STATE_FAILED}})));
    ack.acknowledge();
}
```

## Checklist CI/CD
- [ ] `grep -r "@Transactional" src/` → aucun couvrant plusieurs services/BDD
- [ ] Chaque entité a un état explicite visible : `SELECT status, COUNT(*) FROM {{TABLE}} GROUP BY status`
- [ ] Scheduler de réconciliation actif : alerter si état intermédiaire > {{RECONCILIATION_TIMEOUT}}
- [ ] Test de compensation : événement FAILED → entité annulée (idempotent)
