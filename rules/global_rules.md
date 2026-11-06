# AI CODING RULES: ENFORCED ON EVERY REQUEST

> Place this file at the repository root as `AGENTS.md` (Devin reads it automatically).
> You may also paste it into Devin Knowledge. Rules marked **[BLOCKER]** must never be skipped.

---

## 0. TECH STACK & GROUND RULES

- **Backend:** .NET, any version (.NET Framework 4.x, .NET Core, .NET 5 to latest).
- **Frontend:** Angular, any version.
- **Databases:** SQL Server (T-SQL) and Oracle (PL/SQL).
- **VCS:** Git on Bitbucket and Azure Repos.

**[BLOCKER] Detect, never assume, and never upgrade.**
- Detect the versions in use from `*.csproj`, `Directory.Build.props`, `global.json`, `packages.config`, `package.json`, `angular.json` and `tsconfig.json`.
- Do NOT upgrade .NET, Angular, NuGet or npm packages unless explicitly asked.
- Use only language and framework features available in the detected version (C# language version, TypeScript version, Angular API level).
- Match the existing style, architecture and conventions of the project before introducing new ones.
- If the target framework is legacy (.NET Framework), use `msbuild` / `nuget restore` / `vstest.console` instead of the `dotnet` CLI, and skip rules that need unsupported features (for example, nullable reference types where the C# version does not support them).

**Proportionality:** for trivial changes (typo, config value, one-line fix) the plan and manifest may be 2 to 3 lines. For anything else, follow the full process.

---

## 1. ZERO-WARNING POLICY

Treat every compiler warning, analyzer warning, SonarQube issue, StyleCop issue, nullable warning, ESLint/TSLint error, security hotspot and failing test as a build blocker.
Never ignore a warning, never suppress one without justification, and never leave `TODO` / `FIXME` placeholders.

---

## 2. PLAN BEFORE CODE

Before writing any code, output a numbered plan covering:
1. What the feature or fix does and why
2. Exact technical approach and architecture decisions
3. Algorithm and data flow
4. Dependencies and impacted components (API, UI, DB, other services)
5. Risks, edge cases, performance and security implications
6. Backward compatibility (API contracts, DB schema, existing clients)
7. SonarQube considerations
8. Testing strategy
9. Version bump decision (PATCH / MINOR / MAJOR, see section 11)

Do not write code until the plan is complete.
If the request is ambiguous or may be breaking, ask before proceeding rather than guessing.

---

## 3. FILE MANIFEST

After the plan and before any code, output a manifest in this exact format:

```
📁 FILE MANIFEST
  CREATE  : path/to/new_file.ext
  MODIFY  : path/to/existing_file.ext
  DELETE  : path/to/obsolete_file.ext
```

The manifest must always include the main `.csproj` (version bump) and `CHANGELOG.md`.
Do not touch files that are not in the manifest. If scope grows, update the manifest first.

---

## 4. TEST-DRIVEN DEVELOPMENT (STRICT)

- Write failing tests first. Explain why they fail. Confirm the red phase.
- Do not write production code before tests exist.
- Implement the minimum code to pass the tests, then refactor with tests green.
- Backend: use the test framework already in the project (xUnit / NUnit / MSTest). Do not introduce a new one.
- Frontend: use the project's existing runner (Jasmine/Karma, Jest or Vitest).
- Bug fixes must start with a test that reproduces the bug.

---

## 5. FULL TEST COVERAGE

Every branch and every new line must be covered. Required scenarios:
- Happy paths, validation failures, null/empty inputs, boundary values
- Exception paths, concurrency scenarios, security-sensitive paths
- Retry/fallback logic, `CancellationToken` flows, timeout handling
- Angular: component rendering, user interaction, form validation, service calls (mock `HttpClient`), error states, unsubscribe/cleanup

Tests must be deterministic, isolated, parallel-safe and free of duplication.
Tests must never depend on a real database, external API, system clock or network. Use mocks, fakes, in-memory providers or test containers.
Do not delete or weaken an existing test to make a build pass.

---

## 6. SONARQUBE: ZERO ISSUES

Every change must be free of SonarQube violations before the task is done.

### Hard blockers, never introduce
- **Bugs:** null dereferences, unreachable code, resource leaks, incorrect async usage, infinite recursion, race conditions, deadlocks, blocking async calls, incorrect equality comparisons.
- **Vulnerabilities:** SQL injection, XSS, command injection, insecure deserialization, hardcoded credentials, sensitive data exposure, unsafe reflection, weak cryptography, open redirects, path traversal, SSRF, insecure CORS.
- **Security hotspots:** must be documented with justification and mitigation. Never ignore silently.

### Warnings, must be fixed, not suppressed
Cognitive complexity > 15, methods > 30 lines, > 4 parameters, nesting > 3, duplicate code, dead code, empty catch blocks, magic strings/numbers, nullable warnings, unused variables/imports, async without await, synchronous IO in async flows, poor naming, inconsistent formatting, missing XML docs where required.

### Suppression policy
Suppress only for documented false positives, or when a fix would break required functionality. Every suppression must include justification, risk explanation and mitigation.
Allowed formats: `// NOSONAR: <justification>`, `[SuppressMessage(...)]`, `#pragma warning disable`, `// eslint-disable-next-line <rule> -- <justification>`.
**Never suppress:** security vulnerabilities, injection warnings, hardcoded secrets, null risks, resource leaks, auth issues.

---

## 7. MANDATORY STATIC ANALYSIS: RUN BEFORE COMPLETING ANY TASK

### .NET (SDK-style projects)
1. `dotnet restore`
2. `dotnet build --no-restore -warnaserror`
3. `dotnet test --no-build` (with coverage collection if configured)
4. `dotnet format --verify-no-changes`
5. SonarScanner (if configured)
6. Roslyn analyzers
7. StyleCop analyzers (if enabled)
8. `dotnet list package --vulnerable --include-transitive`

### .NET Framework (legacy)
Use `nuget restore`, `msbuild /warnaserror`, and `vstest.console` or the test runner configured in the repo.

### Angular
1. `npm ci` (never `npm install` unless changing dependencies on request)
2. `ng lint`
3. `ng test --watch=false --browsers=ChromeHeadless` (or the project's equivalent)
4. `ng build --configuration production`
5. `npm audit --audit-level=high`

The task is NOT complete if any step produces warnings, failures or new SonarQube issues.
If a tool is not installed or not configured, say so explicitly in the final report. Never claim a check passed if it was not run.

---

## 8. .NET-SPECIFIC RULES

- Keep nullable reference types enabled (where the C# version supports them).
- Never use `async void` except for event handlers.
- Every async operation must support `CancellationToken` where applicable.
- Use `await` instead of `.Result` or `.Wait()`.
- Dispose `IDisposable` resources via `using` / `await using`.
- Avoid `Task.Run` inside ASP.NET request pipelines.
- Use `ConfigureAwait(false)` in reusable library code (not needed in ASP.NET Core controllers).
- Never swallow exceptions silently. Log with contextual information.
- Never log secrets, passwords, tokens, card data (PAN/CVV) or PII.
- Parameterize all queries (EF Core, ADO.NET, Dapper). Never build SQL by string concatenation or interpolation.
- Validate all external inputs (DTOs, query strings, headers, files).
- Use `nameof()` instead of hardcoded member names.
- Prefer pattern matching over nested `if` chains, and guard clauses over deep nesting.
- Expose collections as `IReadOnlyCollection` / `IReadOnlyList`.
- Avoid reflection, static mutable state and magic configuration keys.
- Prefer dependency injection over manual instantiation. Use `IHttpClientFactory` for HTTP calls.
- Use `IOptions<T>` for configuration. No secrets in `appsettings*.json`. Use environment variables, user-secrets or Key Vault.
- Public API changes must stay backward compatible. Breaking changes require a MAJOR bump and explicit approval.
- Use `decimal` for money, never `float` or `double`. Use UTC for all stored timestamps.
- Payment and financial flows must be idempotent. Never retry a non-idempotent operation blindly.

---

## 9. ANGULAR-SPECIFIC RULES

- Follow the conventions of the detected Angular version (for example, standalone components, signals, and built-in control flow `@if/@for` only if the version supports them. Otherwise use NgModules and `*ngIf/*ngFor`).
- TypeScript `strict` mode stays on. No `any`. Use explicit types, interfaces and `unknown` + type guards.
- Use `ChangeDetectionStrategy.OnPush` for new components where supported.
- Always unsubscribe from observables (`async` pipe, `takeUntilDestroyed`, `takeUntil`). No nested subscriptions. Use RxJS operators (`switchMap`, `mergeMap`, ...).
- Keep components thin. Put logic in services. Use typed reactive forms.
- Never use `innerHTML` / `bypassSecurityTrust*` with unsanitized data.
- Never store tokens or secrets in `localStorage`, source code or environment files committed to Git.
- Put HTTP concerns (auth headers, error handling) in interceptors, not in components.
- Use `trackBy` / `track` in lists. Lazy-load feature routes.
- Do not add npm dependencies without justification. Prefer built-in Angular features.
- Keep i18n and accessibility (labels, ARIA, keyboard navigation) intact.

---

## 10. DATABASE RULES (SQL SERVER & ORACLE)

- Detect which engine a project uses. Never mix T-SQL and PL/SQL syntax. Keep SQL compatible with that engine and with its version.
- **Always use bind parameters** (`@param` for SQL Server, `:param` for Oracle). Never concatenate input into SQL.
- **Schema changes are scripted and versioned**, never ad hoc. Follow the project's migration convention. If none exists, use `YYYYMMDD_HHMM_<short_description>.sql`.
- Every migration must be **idempotent** (safe to re-run) and ship with a **rollback script**.
- Never run destructive statements (`DROP`, `TRUNCATE`, `DELETE`/`UPDATE` without `WHERE`) on any shared or production database. Never connect to production. Only use local/dev/test databases.
- No `SELECT *` in application code. Name columns explicitly.
- Add indexes for new foreign keys and frequent filters. Check execution plans for new heavy queries.
- Use explicit transactions for multi-statement changes, and keep them short.
- SQL Server: prefer `SET NOCOUNT ON` in procedures, use `THROW` rather than `RAISERROR`, avoid scalar UDFs in hot paths.
- Oracle: be explicit about `NUMBER` precision/scale, `VARCHAR2` byte vs char semantics and `DATE` vs `TIMESTAMP`. Mind the 30-character identifier limit on older versions (before 12.2) and avoid empty-string vs `NULL` assumptions (Oracle treats `''` as `NULL`).
- Never store secrets or plain-text sensitive data (card numbers, passwords) in the DB. Use hashing or encryption as appropriate.

---

## 11. VERSIONING: MANDATORY VERSION BUMP

For every task that modifies the repository, increment the version in the **main project's `.csproj`** before the final commit. This applies even if the change is Angular-only or SQL-only, because the .NET project carries the release version.

### Main project mapping

| Solution folder | File to modify |
|---|---|
| `WebServiceMPaymentV3` | `WebServiceMPaymentV3/MpaymentAPI/MpaymentAPI.csproj` |
| *(add other solutions here)* | *(path to main .csproj)* |

### What to change
Inside the first `<PropertyGroup>`, update BOTH tags to the same new value:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <FileVersion>2.1.1</FileVersion>
    <InformationalVersion>2.1.1</InformationalVersion>
  </PropertyGroup>
</Project>
```

### Bump rules
1. Use SemVer `MAJOR.MINOR.PATCH`:
   - **PATCH** (default): bug fix, refactor, tests, docs, config/SQL tweak, Angular UI fix.
   - **MINOR**: new feature, new endpoint, new DB table/column (backward compatible).
   - **MAJOR**: breaking change (API contract, DB schema break). **Ask before doing this.**
2. **Bump once per task/PR, not per commit.** Before bumping, compare with the target branch:
   `git diff origin/<target-branch> -- <path-to-csproj>`.
   If the version is already higher than the target branch, do not bump again.
3. Edit only those two tags. Do not add `AssemblyVersion` / `Version` if absent, do not touch other `.csproj` files, and do not reformat the file (preserve encoding/BOM and line endings).
4. If a tag is missing, add it to the first `<PropertyGroup>` rather than creating a new one.
5. If the main `.csproj` cannot be found, **stop and ask**. Do not guess.
6. The `.csproj` version is the **single source of truth**. The changelog and the commit message must use the exact same version number.

---

## 12. CHANGELOG: UPDATE ON EVERY TASK

1. Find or create `CHANGELOG.md` at the project root.
2. Prepend a new entry using **the same version as the `.csproj`** (section 11):

```
## [2.1.2] - DD-MM-YYYY
### <TYPE>
<TYPE>: [<TAG>] <short description>
```

Types: `FEATURE`, `FIX`, `REFACTOR`, `DOCS`, `TEST`, `SECURITY`, `DB`.
Example:

```
## [2.1.2] - 26-10-2026
### FIX
FIX: [PAYMENT] resolve null reference in wallet payment flow
```

Do not use the old `V1.x.x.x` four-part format. Use the same three-part version as the `.csproj`.
For DB changes, mention the script file name and whether a rollback script exists.

---

## 13. COMPLEXITY: AUTO-REFACTOR WHEN

- Method > 30 lines or cognitive complexity > 15
- Nesting depth > 3
- More than 4 parameters or 3 boolean conditions
- Repeated conditional logic

Refactor order: guard clauses → extract method → extract class → strategy → specification → pipeline.
Refactor only code touched by the task. Do not rewrite unrelated legacy code unless asked.

---

## 14. DRY: DEDUPLICATION PASS REQUIRED

Before finalizing, flag and eliminate:
- Identical or near-identical logic blocks (3+ lines)
- Repeated validation, mapping or exception handling
- Duplicate constants, LINQ expressions or class logic (> 50% shared)

Remediation order: extract method → extract helper → extract class → shared constants → generic abstraction → pattern.

---

## 15. SECURITY & SECRETS [BLOCKER]

- Never commit or print secrets, connection strings with passwords, API keys, tokens, certificates or private keys.
- Use placeholders (`<your-value>`) in examples, config templates and tests.
- Never log or return stack traces, SQL or internal details to API clients.
- Apply least privilege: authentication and authorization on every endpoint. Explicitly mark anonymous endpoints.
- Validate and sanitize all input. Encode all output.
- Use TLS only, and never disable certificate validation.
- Do not add a dependency with known high/critical vulnerabilities. Check licenses for new packages.

---

## 16. GIT WORKFLOW (BITBUCKET / AZURE REPOS)

- **Never push directly to `main`, `master`, `develop` or release branches.** Always work on a feature branch and open a pull request.
- Branch naming: `feature/<ticket>-short-desc`, `fix/<ticket>-short-desc`, `chore/<short-desc>`, `hotfix/<ticket>-short-desc`.
- Rebase or merge the target branch before opening the PR, and resolve conflicts carefully. Never force-push to shared branches.
- Keep commits small and focused. One logical change per commit.
- Commit message format: `<type>(scope): short description`
  Types: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`, `perf`, `build`, `ci`.
  Examples: `feat(wallet): add retry policy`, `fix(api): resolve null ref in payment flow`, `chore(release): bump version 2.1.1 -> 2.1.2`.
- The PR description must include: summary, plan, files changed, test results, version bump (`old -> new`), DB scripts (if any), and risks/rollback.
- **Never commit:** `bin/`, `obj/`, `TestResults/`, `coverage/`, `.sonarqube/`, `node_modules/`, `dist/`, `.angular/`, logs, temp files, secrets, `.env`, `*.user`, or generated artifacts.
- Do not rewrite or delete Git history.

---

## 17. CODE QUALITY PREFERENCES

**Prefer:** small focused methods, immutable models, composition over inheritance, dependency injection, explicit typing for clarity, early validation, pure functions, domain-driven naming, single responsibility, clean architecture boundaries, structured logging.

**Avoid:** god classes, hidden side effects, static mutable state, tight coupling, comments that explain bad code, premature optimization, over-engineering, unrelated "drive-by" changes.

---

## 18. DEVIN WORKING AGREEMENT

- Stay strictly within the scope of the request. Do not make unrelated changes.
- If requirements are unclear or risky (breaking change, data loss, production impact), stop and ask before acting.
- Do not invent APIs, packages, endpoints or table/column names. Verify they exist in the codebase first.
- If you cannot run a required check (missing tool, no network, no DB), say so clearly. Never fake results.
- When blocked after a reasonable number of attempts, report what was tried and what is needed rather than looping.
- Final report must be honest and complete (see below).

### Final report format
```
✅ SUMMARY      : what was done and why
📁 FILES        : created / modified / deleted
🔢 VERSION      : old -> new (path to .csproj)
🧪 TESTS        : added/updated, pass/fail counts, coverage
🔍 ANALYSIS     : build / format / Sonar / lint results (or "not run: reason")
🗄️ DB SCRIPTS   : scripts + rollback (or "none")
⚠️ RISKS/NOTES  : limitations, follow-ups, suppressions with justification
💬 COMMIT       : proposed commit message
```

---

## TASK COMPLETION CHECKLIST: ALL MUST BE TRUE

- [ ] Plan and file manifest produced before coding
- [ ] Tests written first and passing (backend and frontend as applicable)
- [ ] Build successful with zero warnings
- [ ] Zero analyzer / lint warnings
- [ ] Zero SonarQube issues
- [ ] Zero formatting violations
- [ ] Deduplication pass completed
- [ ] **Version bumped in the main `.csproj` (`FileVersion` + `InformationalVersion`), once per task**
- [ ] **Changelog updated with the same version**
- [ ] DB scripts are idempotent with rollback (if applicable)
- [ ] No secrets, debug artifacts or build output committed
- [ ] No undocumented suppressions
- [ ] Work is on a feature branch, no direct push to protected branches
- [ ] Commit message prepared and PR description complete
- [ ] Final report delivered