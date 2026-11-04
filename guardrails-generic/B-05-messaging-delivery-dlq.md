---
id: "B-05"
titre: "Messaging — Delivery garantie, DLQ, Domain Events, chiffrement en transit"
severite: "🔴 BLOCAGE (événement à effet at-most-once / sans DLQ / en clair) — 🟡 ALERTE"
applicable_si: [backend]
declencheurs:
  - "Tout producer/consommateur de messages : {{MESSAGE_BROKER}} (Kafka, RabbitMQ, SQS, etc.)"
  - "Tout appel inter-service synchrone remplaçable par un événement de domaine"
  - "Tout topic/queue sans chiffrement sur un flux sensible ({{COMPLIANCE_STANDARD}})"
version: "1.0"
---

## Risque
`{{DELIVERY_GUARANTEE}}=at-most-once` sur un topic à effet : message perdu = opération perdue sans trace.
Chiffrement absent sur un topic sensible : violation de {{COMPLIANCE_STANDARD}} (données en transit non chiffrées).
Appel HTTP synchrone inter-service : si le service cible est down → la transaction entière échoue → couplage temporel.

## Règles

**Delivery & DLQ :**
- Topic à effet → `{{ACKS_CONFIG}}=all` + `enable.idempotence=true` (ou équivalent selon {{MESSAGE_BROKER}}).
- Consumer à effet → commit **manuel** de l'offset **après** l'effet idempotent (fiche B-02).
- Message poison → N échecs → Dead Letter Queue `{{DLQ_TOPIC}}` + alerte.
- Topics critiques : `replicationFactor: {{REPLICATION_FACTOR}}`, `minInsyncReplicas: {{MIN_ISR}}`.

**Domain Events (découplage) :**
- Appel HTTP synchrone pour notifier un autre service → remplacer par un événement de domaine sur {{MESSAGE_BROKER}}.
- Pattern : service A publie `{{EVENT_NAME}}` → service B consomme (at-least-once + idempotence).

**Chiffrement en transit :**
- Topic sensible → `{{SECURITY_PROTOCOL}}=TLS` ou équivalent — jamais en clair.
- Credentials → gestionnaire de secrets (fiche S-04).

## Anti-pattern
```java
// ❌ BLOCAGE — producer sans garantie de delivery
messagingClient.send("{{TOPIC}}", event); // acks=1 par défaut

// ❌ BLOCAGE — commit avant l'effet
void handle({{EVENT}} event) {
    // commit automatique AVANT l'effet → double traitement si crash
    service.process(event);
}

// ❌ BLOCAGE — topic sensible en clair
{{SECURITY_PROTOCOL}}: PLAINTEXT
```

## Pattern conforme
```java
// ✅ Producer à effet — delivery garantie
@Bean
ProducerFactory<String, Object> producerFactory() {
    Map<String, Object> config = new HashMap<>();
    config.put(ProducerConfig.ACKS_CONFIG, "all");
    config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "true");
    config.put(ProducerConfig.SECURITY_PROTOCOL_CONFIG, "{{SECURITY_PROTOCOL}}");
    return new DefaultKafkaProducerFactory<>(config);
}

// ✅ Consumer — commit manuel APRÈS l'effet
void handle({{EVENT}} event, Acknowledgment ack) {
    if (!repository.existsBy(event.id())) { // idempotence (fiche B-02)
        service.process(event);
    }
    ack.acknowledge(); // offset commité APRÈS l'effet
}
```

## Checklist CI/CD
- [ ] `grep -r "{{SECURITY_PROTOCOL}}" config/ | grep "PLAINTEXT"` → 0 sur topics sensibles
- [ ] Test : kill du consumer en cours → rejeu → 1 seul effet (at-least-once + idempotence)
- [ ] Test : message poison → DLQ `{{DLQ_TOPIC}}` + alerte < 30 s
- [ ] Test : connexion sans TLS → rejet
- [ ] Topics à effet : replicationFactor={{REPLICATION_FACTOR}}, minInsyncReplicas={{MIN_ISR}}
