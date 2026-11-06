# Playbook: !guardrails-review

## Overview
Audit a pull request or a set of files against the team guardrails. Report only; do not change code unless the user asks.

## Procedure
1. Identify the changed files (PR diff or the paths given). Ask for the PR link or paths if none were given.
2. Classify the change (security, money, endpoint, persistence, migration, messaging, Angular, tests) and read the matching file(s) in docs/guardrails/.
3. For each rule that applies, check the code and run the rule's CI checklist commands (grep, dotnet test, dotnet stryker, npm run lint) where they can run in this environment.
4. Report each violation as: [ID] 🔴 BLOCKER | 🟡 WARNING — Problem, Rule, Fix (corrected code), Checklist.
5. Separate facts (what the code does) from interpretations (assumed impact). If information is missing (database engine, broker, configuration), list it as an open question; do not guess.
6. Finish with a summary: counts of blockers and warnings, rules that were checked and passed, and rules that could not be checked and why.

## Specifications
- Never modify code, analyzers, architecture tests or thresholds during a review.
- Never include real secrets or sensitive values in the report.
- Findings must cite file path and line.

## Advice
- Review one PR at a time; large diffs give shallow results.
- Use the 🔴 findings as merge blockers and confirm them by running the checklist command.

---

# Playbook: !guardrails-implement

## Procedure
1. Before writing code, read the guardrails file for each area the task touches (see AGENTS.md table).
2. Write the failing test first for any business rule (exact value and exact boundary).
3. Implement; run `dotnet build -warnaserror`, `dotnet test`, and frontend lint/tests.
4. Run the relevant checklist commands; fix violations.
5. In the PR description list guardrails applied and any that could not be satisfied.

## Specifications
- Do not silence analyzers, skip architecture tests or lower mutation thresholds.
- Do not touch the Flutter app.
