---
id: "S-01"
titre: "SBOM, licences & CVE scan"
severite: "🔴 BLOCAGE (copyleft fort / CVE ≥ 9.0) — 🟡 ALERTE (inconnue / CVE 7-8.9)"
applicable_si: [backend, frontend, mobile]
declencheurs:
  - "Ajout/mise à jour de dépendance : {{PACKAGE_MANAGER_FILE}} (pom.xml, package.json, go.mod, Cargo.toml)"
  - "Rapport Trivy/Grype/Dependabot signalant une CVE sur une dépendance de {{MODULE_CIBLE}}"
version: "1.0"
---

## Risque
Une dépendance AGPL/GPL contaminante oblige à ouvrir tout le code propriétaire de {{MODULE_CIBLE}} sous la même licence.
Une dépendance avec CVE CVSS ≥ 9.0 est exploitable dès le merge — aucun contrôle applicatif ne protège contre une RCE côté librairie.

## Règles

- **Licences bloquées** : AGPL, GPL, LGPL sans exception écrite, propriétaire fermée, SSPL.
- **Licences autorisées** : MIT, Apache 2.0, BSD, ISC, MPL 2.0 (isolation module).
- **CVE CVSS ≥ 9.0** → BLOCAGE : patcher ou remplacer avant merge.
- **CVE CVSS 7.0–8.9** → ALERTE : patch dans la sprint courante, justification TL requise.
- **CVE < 7.0** → trace, suivi en backlog.
- Jamais de version `LATEST` ou `*` sur une dépendance critique de {{MODULE_CIBLE}}.

## Anti-pattern
```
# ❌ BLOCAGE — licence GPL contaminante dans {{MODULE_CIBLE}}
dependance: some-gpl-lib:2.1  # licence GPL non compatible avec licence propriétaire

# ❌ BLOCAGE — CVE CVSS 9.8 connue
dependance: log4j-core:2.14.1  # CVE-2021-44228 (Log4Shell)
```

## Pattern conforme
```bash
# ✅ Vérification CI obligatoire sur {{MODULE_CIBLE}}
trivy fs --severity HIGH,CRITICAL --exit-code 1 .
grype . --fail-on high

# ✅ Rapport de licences
license-checker --failOn "GPL;AGPL;LGPL"
```

## Checklist CI/CD
- [ ] `trivy fs --severity CRITICAL --exit-code 1` → 0 résultats en CI sur {{MODULE_CIBLE}}
- [ ] Toute nouvelle dépendance auditée : licence dans la liste autorisée
- [ ] CVE CVSS ≥ 9.0 → PR bloquée jusqu'au patch
- [ ] Dépendances publiées il y a ≥ 7 jours (protection supply chain)
