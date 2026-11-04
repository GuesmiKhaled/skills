---
name: docs-generator
description: |
  Creates task-oriented technical documentation with progressive disclosure. Use when writing READMEs, API docs, architecture docs, or Markdown documentation.
  Also use this skill at the END of any completed reverse engineering, penetration testing, CTF, or security analysis task to generate a formal report in the user's project directory.
  Trigger keywords: write a report, write documentation, produce a report, writeup, technical documentation, report, documentation.
---

# Technical documentation

## ACTION REQUIRED (execute immediately after reading)

1. `NOW`: Confirm whether the current task falls within this skill's scope.
2. `NOW`: Read `../tool-index.md` to verify tool availability and actual paths.
3. `NEXT`: If tools are missing, invoke bootstrap. Do not guess paths.
4. `ACT`: Proceed to the first step of the workflow and execute it. Do not stop at the confirmation stage.

For writing style, tone, and voice guidance, use `Skill(ce:writer)` with the **The Engineer** persona.

## Documentation output for security and reverse engineering tasks

After a reverse engineering, penetration testing, CTF, or security analysis task is completed, this skill is responsible for generating formal technical documentation in the **user's project directory**.

### When to trigger

1. A reverse engineering task is complete and its core findings have been established, such as algorithm reconstruction, signature analysis, or a bypass technique.
2. A penetration test is complete and vulnerabilities have been discovered and verified.
3. A CTF challenge has been solved and the flag has been obtained.
4. The user explicitly requests a report, documentation, or a writeup.

### Template selection

| Task type | Template |
|---|---|
| APK / binary / shared-library (`.so`) reverse engineering | `references/security-report-templates.md` → Reverse Engineering Report |
| Penetration testing / vulnerability research | `references/security-report-templates.md` → Penetration Testing Report |
| CTF challenge writeup | `references/security-report-templates.md` → CTF Writeup |
| JavaScript / web signature reverse engineering | `references/security-report-templates.md` → Signature Reverse Engineering Report |
| Malware / APT / virus analysis report | `references/security-report-templates.md` + **`references/vendor-report-rules.md`** |
| General technical documentation | `references/templates.md` → README / API Documentation |

### Vendor-style report structure (Issue #65)

Formal security reports **MUST** read `references/vendor-report-rules.md` for structure only; do not copy vendor source text. Select a vendor-specific flavor only when supported by task evidence or explicitly requested by the user. Use `flavor = null` for ordinary reverse engineering and other tasks.

| Flavor / overlay | When to use | Primary reference structure |
|---|---|---|
| `malware` | Explicitly malicious samples, trojans, signed legitimate binaries used to load malicious code, phishing, or poisoning | Huorong-style: Overview → Execution flow → Sample analysis → Incident response → IOCs |
| `apt` | APT activity, campaigns, threat groups, multi-stage infection chains, or industry-targeted attacks | Kaspersky Securelist-style: Executive summary → Infection chain → Investigation narrative → Interesting findings → Technical analysis → Detection and mitigation → IOCs |
| `flavor = null` | Ordinary APK / ELF / PE / Mach-O reverse engineering, algorithm or firmware analysis, penetration testing, CTF, or JavaScript signature analysis | Original task template + applicable Base elements; do not impose malware- or APT-specific sections |
| Thin `vuln` overlay | The user explicitly requests technical analysis of a vulnerability, patch, or CVE | Overview → Impact / reproduction → Crash and patch analysis → Mitigation recommendations; layered onto `null`, not a third default full-report flavor |

Principle: **Templates should be precise, not numerous.** Maintain only two full vendor-style flavors. `vuln` is an optional thin overlay, not a separate third default full-report template.

This rule applies **alongside** the §0 Evidence → Finding → Path requirement. If they conflict, the Evidence contract takes precedence.

### Output requirements

- **Output location:** The user's current project directory, not the skill package directory.
- **Filename format:** `YYYY-MM-DD_[type]-[target-short-name]-report.md`
- **If the project has a `docs/` directory:** Prefer placing the report there.
- **Encoding:** UTF-8.
- **Language:** Follow the language of the conversation. Produce Chinese reports for Chinese conversations and English reports for English conversations.

### Quality requirements

- All code blocks must be directly executable or include clear contextual instructions.
- Do not include placeholders or TODOs.
- Key findings must be supported by evidence.
- Reproduction steps must allow an independent third party to reproduce the results.
- Replace sensitive information, including real tokens, passwords, and internal URLs, with placeholders.
- **MUST** include an Evidence → Finding → Path chain (see `../ops/evidence-finding-path.md` and §0 of the templates).
- **MUST** read `references/vendor-report-rules.md` and select `malware`, `apt`, or `flavor = null`. Vulnerability tasks may add the thin `vuln` overlay. Without a vendor-specific flavor, include only the original task template and applicable Base elements; do not force IOCs or MITRE ATT&CK content.
- **SHOULD** reference the case's `scope.md` and `timeline.md` files, if available (`../scripts/case-init.ps1`).

### Diagram integration

When generating a report, invoke the `diagram-generator` skill to create visual diagrams where appropriate.

| Report type | Recommended diagrams | Diagram types |
|---|---|---|
| Reverse engineering report | Function call graph, data-flow diagram | Mermaid flowchart / sequence diagram |
| Penetration testing report | Attack path, network topology | Mermaid flowchart / Graphviz |
| CTF writeup | Problem-solving workflow | Mermaid flowchart |
| JavaScript signature reverse engineering report | Request-sequence diagram, algorithm workflow | Mermaid sequence diagram / flowchart |

Embed diagrams as Mermaid code blocks in the report's Markdown so they render directly on GitHub and GitLab.

---

## Core principles

### 1. Progressive disclosure

Reveal information in layers:

| Layer | Content | User question |
|---|---|---|
| 1 | One-sentence description | What is it? |
| 2 | Quick-start code block | How do I use it? |
| 3 | Full API reference | What are my options? |
| 4 | Architecture deep dive | How does it work? |

**Warnings, breaking changes, and prerequisites belong at the TOP.**

### 2. Task-oriented writing

```markdown
<!-- Bad: Feature-oriented -->
## AuthService class

The AuthService class provides authentication methods...

<!-- Good: Task-oriented -->
## Authenticating users

To authenticate a user, call `login()` with credentials:
```

### 3. Show, don't tell

Every concept needs a concrete example.

## Formatting standards

- **Sentence-case headings:** Use "Getting started", not "Getting Started".
- **Maximum of three heading levels:** Split deeper content into separate documents.
- **Always specify the language** in code blocks.
- **Use relative paths** for internal links.
- **Use tables** for structured data with three or more attributes.

## Quality checklist

- [ ] Code examples have been tested and are runnable.
- [ ] No placeholder text or TODOs.
- [ ] Documentation matches actual code behavior.
- [ ] The document is scannable without requiring the reader to read everything.
- [ ] The reader knows what to do next.

## Anti-patterns

| Problem | Fix |
|---|---|
| Wall of text | Break content into headings, bullet points, and tables. |
| Critical information buried in the document | Put warnings and breaking changes at the top. |
| Missing error documentation | Always document what can go wrong. |

## Templates

For README, API endpoint, and file-organization templates, see [`references/templates.md`](references/templates.md).

## Related skills

- `Skill(ce:writer)` — Writing style, tone, and voice; load the **The Engineer** persona.
- `Skill(ce:visualizing-with-mermaid)` — Architecture and flow diagrams.

---

## On-demand bootstrap

This skill does not depend on external tools and generates plain text only. Bootstrap is not required.

If diagrams need to be rendered and embedded in a report, invoke the `diagram-generator/` skill.

---

## Routing context

**Upstream entry point:** All security and reverse engineering skills automatically invoke this skill when their tasks are completed.

**Trigger methods:**
- **Automatic:** Execute as step 9 of the workflow chain after task completion.
- **Manual:** The user says "write a report", "produce documentation", or "writeup".

**Related peer modules:**
- `apk-reverse/` — Generate a reverse engineering report after APK analysis.
- `ida-reverse/` — Generate a reverse engineering report after binary analysis.
- `radare2/` — Generate a report after CLI-based analysis.
- `js-reverse/` — Generate a signature analysis report after JavaScript reverse engineering.
- `reverse-engineering/` — Generate a report after general reverse engineering.
- `field-journal/` — Use report content as a data source for the evolution log.

**Security report templates:** `references/security-report-templates.md`

**Vendor report rules:** `references/vendor-report-rules.md` (`flavor`: `malware` | `apt` | `null`; optional overlay: `vuln`)

**General documentation templates:** `references/templates.md`

## Completion self-check (MUST pass before claiming completion)

- [ ] Did I execute every workflow step rather than merely read it?
- [ ] Did I use actual tool paths verified against `tool-index`?
- [ ] Did I produce reproducible evidence, such as commands, scripts, screenshots, or reports?
- [ ] Does the report contain the Evidence / Finding / Path chain required by the operations contract?
- [ ] Have I completed and written back every checklist item required by the RULES?
