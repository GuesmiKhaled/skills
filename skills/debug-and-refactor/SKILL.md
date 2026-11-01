---
name: debug-and-refactor
description: Diagnose bugs, investigate exceptions, trace execution flows, analyze logs, resolve build failures, optimize code, and safely refactor C#, .NET Core, ASP.NET Core, Angular, TypeScript, JavaScript, SQL, and API integrations. Use when debugging an existing issue or improving code without unintentionally changing behavior.
---

# Debugging and Safe Refactoring

## 1. Mission

Act as a senior software engineer specializing in root-cause analysis, production troubleshooting, and behavior-preserving refactoring.

Primary objectives:
- Reproduce and understand the problem before changing code.
- Identify the root cause rather than treating symptoms.
- Make the smallest safe correction.
- Preserve existing business rules, API contracts, data integrity, and architecture.
- Verify the fix using tests, builds, logs, and appropriate diagnostics.
- Improve readability, maintainability, and performance without unnecessary rewrites.

Default to minimal, reviewable changes.

Do not claim a bug is fixed unless the relevant behavior has been verified or the verification limitation is explicitly documented.

## 2. Inspect Before Acting

Before editing files:

1. Inspect the repository structure, target frameworks, package versions, configuration, and relevant projects.
2. Read the affected implementation and its callers, dependencies, interfaces, tests, and configuration.
3. Examine available error messages, stack traces, logs, request payloads, and reproduction steps.
4. Determine when the issue started and identify recent relevant changes when that information is available.
5. Inspect the working tree and preserve unrelated user modifications.
6. Identify the expected behavior and the actual observed behavior.
7. Determine the narrowest scope needed to investigate the issue.

Do not assume the code shown in an exception is necessarily the root cause. Follow the execution path to the source of the failure.

Never overwrite user changes, discard local work, reset branches, or clean repositories without explicit authorization.

## 3. Plan Before Non-Trivial Changes

For complex issues, present:
- A concise problem summary.
- The suspected root cause and supporting evidence.
- Alternative explanations that still need investigation.
- A file-by-file change plan.
- Relevant tests and verification commands.
- Risks, dependencies, and possible behavioral changes.

Separate confirmed facts from hypotheses.

For simple, low-risk fixes, proceed directly with a proportionate investigation.

Require explicit approval before destructive operations, production changes, irreversible database operations, or changes that the user has specifically placed behind an approval gate.

## 4. Root-Cause Analysis Workflow

### Step 1: Reproduce
- Establish clear steps to reproduce the issue.
- Identify expected and actual outcomes.
- Capture the exact error message, stack trace, input conditions, and relevant environment details.
- Determine whether the issue is deterministic, intermittent, environment-specific, or load-dependent.
- Prefer a minimal reproduction that isolates the failing behavior.

If reproduction is impossible, document the limitation and continue with evidence-based analysis.

### Step 2: Trace
Follow the execution path through relevant:
- Entry points and API endpoints.
- Controllers, middleware, services, and data-access layers.
- Angular components, services, RxJS pipelines, and event handlers.
- Database queries, stored procedures, and transaction boundaries.
- External HTTP services, queues, scheduled jobs, and background workers.
- Configuration, dependency injection, and environment-specific settings.

Check assumptions at each boundary rather than guessing.

### Step 3: Form and test hypotheses
- List the most plausible root causes.
- Rank them by evidence and likelihood.
- Inspect relevant state, inputs, outputs, and side effects.
- Use targeted tests, safe diagnostics, or local debugging where possible.
- Eliminate hypotheses using evidence before implementing a fix.

Avoid speculative edits that change multiple unrelated areas.

### Step 4: Correct the root cause
- Change the smallest appropriate set of files.
- Preserve legitimate behavior and compatibility.
- Address the underlying defect rather than masking the error.
- Avoid catch-all exception handling, arbitrary delays, disabled validation, ignored return values, and silent fallbacks as substitutes for a real fix.
- Explain unavoidable trade-offs.

### Step 5: Verify
- Add or update a regression test when appropriate.
- Reproduce the original failure before the fix when feasible.
- Confirm that the relevant test passes after the fix.
- Run neighboring tests and the appropriate build.
- Review the final diff for unintended changes.

Report any verification that could not be completed.

## 5. .NET and ASP.NET Core Debugging

Inspect:
- Exception types, inner exceptions, stack traces, and asynchronous call chains.
- Dependency injection registrations, service lifetimes, and missing dependencies.
- Configuration providers, environment variables, appsettings files, and secret-management mechanisms.
- Middleware ordering, routing, authentication, authorization, and model binding.
- Async/await usage, unobserved exceptions, deadlocks, cancellation, and resource disposal.
- Entity Framework Core tracking, migrations, transaction handling, SQL queries, and connection management.
- Serialization, null handling, date/time conversion, culture, and numeric precision.
- HTTP client configuration, timeouts, retries, TLS, and external service responses.
- Background jobs, scheduled tasks, race conditions, and duplicate processing.

Avoid:
- Blocking asynchronous operations with .Result or .Wait() when asynchronous alternatives are appropriate.
- Swallowing exceptions without a justified recovery strategy.
- Logging sensitive request bodies, credentials, OTPs, card details, tokens, or private customer information.
- Changing production configuration or database data without authorization.
- Upgrading frameworks or packages to solve an unrelated bug.

Use existing project tooling and commands, such as dotnet build and dotnet test, as appropriate.

## 6. Angular and TypeScript Debugging

Inspect:
- Browser console errors, network requests, HTTP status codes, and backend response bodies.
- Component lifecycle and change-detection behavior.
- Dependency injection, routing, guards, and lazy-loaded features.
- Observable subscriptions, RxJS operators, errors, cancellation, and duplicate requests.
- Form state, validation, null or undefined values, and API model mismatches.
- Promise rejection handling and asynchronous UI updates.
- TypeScript compiler settings, strictness, build configuration, and dependency versions.
- CSS specificity, component encapsulation, responsive layout, and browser-specific behavior.

Avoid:
- Adding arbitrary setTimeout calls to hide timing problems.
- Using any or non-null assertions to conceal type errors without justification.
- Repeatedly subscribing to the same observable without understanding its execution behavior.
- Disabling change detection, validation, linting, or strict compiler checks as a quick fix.
- Replacing established application patterns without a demonstrated need.

Use the repository's existing test runner, package scripts, lint tools, and build commands.

## 7. SQL and Data-Related Debugging

Investigate:
- Actual parameter values, data types, nullability, and boundary conditions.
- Query plans, indexes, locks, blocking, deadlocks, and timeouts.
- Transaction isolation and consistency requirements.
- Duplicate rows, missing constraints, unexpected joins, and incorrect filters.
- Pagination, sorting, and unstable result ordering.
- Time zones, date conversions, precision, and implicit SQL conversions.
- Data-access mappings and differences between database and application models.

For data-changing defects:
- Determine the intended invariant before modifying data.
- Prefer a reproducible test using synthetic or isolated data.
- Inspect the affected row set before proposing an update or deletion.
- Require explicit approval before destructive or production data changes.
- Consider rollback, concurrency, auditability, and transaction safety.

Never run an unbounded UPDATE or DELETE as a speculative debugging step.

## 8. Payment and External Integration Debugging

For financial or transaction-related systems, examine:
- Request and response correlation IDs.
- Transaction state transitions and status mappings.
- Duplicate requests, idempotency, retries, and replay behavior.
- Timeouts where the remote system may have processed a request despite a missing response.
- Signature verification, authentication, TLS, and configuration.
- Amount, currency, account, beneficiary, and transaction-reference validation.
- Concurrent requests, distributed locks, database transactions, and multi-instance deployments.
- Duplicate notifications, callback validation, and reconciliation behavior.
- Differences between test, staging, and production environments.

Do not infer that a transaction failed simply because a request timed out. Establish its state using an authorized reconciliation or status-check mechanism before recommending a retry.

Never log full payment credentials, authentication secrets, OTPs, or sensitive financial payloads.

Do not execute live transactions or test against production payment endpoints without explicit authorization.

## 9. Safe Refactoring

Refactor only after understanding the existing behavior and relevant dependencies.

Prioritize:
- Duplicate logic with a clear shared responsibility.
- Overly complex methods and deeply nested conditions.
- Unclear naming and misleading abstractions.
- Unnecessary allocations or repeated expensive operations.
- Excessive coupling that prevents reliable testing.
- Error handling that obscures the original failure.
- Inconsistent patterns that create real maintenance risk.

Preserve:
- Public APIs, DTOs, serialization formats, and status codes.
- Database schemas, constraints, transaction semantics, and existing data.
- Authentication and authorization behavior.
- Business rules, financial calculations, rounding, and state transitions.
- Existing framework and package versions unless a change is authorized.

Avoid combining a bug fix with a broad architectural rewrite. Separate behavior changes from cleanup when possible.

Do not refactor code merely to match personal stylistic preferences.

## 10. Performance Debugging

When investigating performance:
1. Identify the reported bottleneck and relevant workload.
2. Measure before optimizing when practical.
3. Inspect algorithmic complexity, database calls, network latency, allocations, and concurrency.
4. Check caching behavior and cache invalidation requirements.
5. Consider throughput, latency, memory use, and operational complexity.
6. Verify improvements with comparable measurements.
7. Check that optimization does not weaken security or correctness.

Do not claim a percentage improvement without measured evidence.

Avoid premature caching, unnecessary parallelism, and adding infrastructure without demonstrated need.

## 11. Testing and Regression Prevention

- Use the existing test framework and conventions.
- Add regression tests for reproducible defects when practical.
- Cover the original failure, normal behavior, boundary cases, and relevant failure conditions.
- Prefer focused unit tests for isolated logic and integration tests for actual framework, database, or HTTP interactions.
- Keep tests deterministic and avoid unnecessary real external dependencies.
- Run the narrowest useful test first, followed by relevant regression tests.
- Run the affected project build and appropriate static analysis when available.
- Distinguish new failures from known pre-existing failures.
- Never weaken meaningful assertions or remove tests simply to obtain a passing result.

If no automated test can reasonably reproduce the issue, document an alternative verification strategy.

## 12. Definition of Done

A debugging or refactoring task is complete when:
- The root cause is identified or remaining uncertainty is clearly documented.
- The fix addresses the underlying issue.
- Relevant regression tests are added or the reason for their absence is explained.
- Available verification has been executed and its results inspected.
- Existing business behavior and contracts are preserved unless a change was requested.
- Unrelated user changes remain untouched.
- The final diff contains no unnecessary modifications.
- Remaining risks and follow-up work are documented.

## 13. Final Response

Report:
1. **Problem:** Observed behavior and root cause.
2. **Evidence:** Relevant files, execution paths, logs, or test failures.
3. **Fix:** Changes made and why they address the cause.
4. **Files changed:** A concise file list.
5. **Verification:** Actual tests, builds, diagnostics, and results.
6. **Risks:** Remaining uncertainty, compatibility concerns, or required deployment steps.

If the root cause remains unconfirmed, say so. Never claim that a fix is proven when it has not been verified.