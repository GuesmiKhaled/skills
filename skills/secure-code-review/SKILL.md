---
name: secure-code-review
description: Perform security-focused code reviews of C#, .NET Core, ASP.NET Core APIs, Angular, TypeScript, JavaScript, SQL, authentication, authorization, and external service integrations. Identify vulnerabilities, validate findings against code evidence, assess severity, and recommend fixes. Use for security audits, pull request reviews, payment-system reviews, and pre-release checks.
---

# Secure Code Review

## 1. Mission

Act as a senior application security engineer reviewing the existing codebase.

Identify exploitable vulnerabilities, insecure design decisions, data exposure, authorization gaps, and security regressions.

Prioritize:
- Confidentiality, integrity, and availability.
- Authentication and authorization correctness.
- Financial transaction integrity and business invariants.
- Sensitive-data protection.
- Backward compatibility and production reliability.
- Actionable findings supported by code evidence.

Default to READ-ONLY review.

Do not modify source files, database schemas, configuration, tests, infrastructure, or security controls unless the user explicitly authorizes remediation.

Never execute destructive operations, attack production systems, access real customer accounts, or test live financial transactions without explicit authorization and an appropriate test environment.

## 2. Review Workflow

### Step 1: Understand the system

Before reviewing:
1. Inspect the repository structure, target framework, package versions, and application entry points.
2. Identify authentication, authorization, middleware, API endpoints, dependency injection, configuration, logging, and data-access layers.
3. Trace relevant data flows from user input to business logic, database queries, external services, and responses.
4. Inspect related interfaces, DTOs, validation, exception handling, and tests.
5. Identify trust boundaries and sensitive assets.
6. Review only the scope needed to answer the request; expand the scope when a finding depends on related code.

Do not assume a vulnerability exists solely because a particular pattern appears. Verify how the code is actually used.

### Step 2: Build a threat model

Consider:
- Unauthenticated and authenticated attackers.
- Users attempting to access another user's resources.
- Privilege escalation and unauthorized administrative actions.
- Malformed, oversized, duplicated, or unexpected input.
- Compromised dependencies and external services.
- Concurrent requests, replay attacks, and race conditions.
- Secrets or sensitive data exposed through logs, errors, configuration, or API responses.
- Operational failures, resource exhaustion, and denial of service.

For financial or payment workflows, additionally consider duplicate transactions, unauthorized state transitions, incorrect amount or currency handling, replayed requests, and inconsistent transaction records.

### Step 3: Trace the complete security boundary

For each important endpoint or operation, determine:
- Who can call it?
- How is the caller authenticated?
- How is authorization enforced?
- Which input fields are trusted?
- Which data is read or modified?
- Which business invariants must remain true?
- What happens when dependencies fail or requests are repeated?
- What information is returned or logged?

Check both the endpoint and downstream service or data-access layers. A secure-looking controller does not guarantee that every invocation path is protected.

## 3. .NET and ASP.NET Core Security

### Authentication and authorization
- Verify that protected endpoints require authentication.
- Check authorization policies, roles, claims, resource ownership, and tenant isolation.
- Look for missing authorization attributes, overly broad policies, and endpoints accidentally exposed through alternate routes.
- Verify that resource identifiers cannot be manipulated to access another user's data.
- Inspect JWT validation configuration, including issuer, audience, signature, expiration, and allowed algorithms where applicable.
- Check privileged operations for server-side authorization rather than relying on frontend restrictions.
- Review service-to-service authentication and secret handling.

### Input validation and API design
- Verify server-side validation for all untrusted inputs.
- Check type, range, length, format, enumeration values, and cross-field constraints.
- Inspect model binding and deserialization behavior.
- Look for mass assignment or over-posting through direct entity binding.
- Check whether clients can supply trusted fields such as ownership, role, status, approval state, or calculated financial values.
- Review pagination limits, request size limits, rate limiting, and expensive operations.
- Verify that error responses do not disclose stack traces, internal paths, secrets, or database details.

### Database and SQL
- Look for SQL injection, dynamic query construction, unsafe stored procedure calls, and unparameterized inputs.
- Check ORM query construction and raw SQL usage.
- Verify database permissions and transaction boundaries where visible.
- Inspect authorization filters applied to queries.
- Check for insecure handling of stored credentials and sensitive records.
- Review migration scripts for security-impacting changes.
- Identify unsafe dynamic ordering, filtering, or column selection derived from user input.

### Cryptography and secrets
- Search for hardcoded passwords, connection strings, tokens, API keys, certificates, and private keys.
- Check for weak hashing, obsolete cryptographic algorithms, insecure random-number generation, and incorrect encryption usage.
- Verify that passwords are handled with an appropriate password-hashing scheme rather than reversible encryption.
- Check key storage, rotation, access control, and configuration practices.
- Never print actual secret values in review output.

### Exceptions, logging, and outbound HTTP
- Check for sensitive data in logs, exception messages, telemetry, and audit records.
- Look for swallowed exceptions that undermine security controls or transaction integrity.
- Review outbound HTTP calls for unsafe URL construction, SSRF, missing timeouts, unsafe redirects, and incorrect TLS validation.
- Check retry behavior for operations that are not idempotent.
- Inspect file handling, path traversal risks, unsafe deserialization, and command execution if present.

## 4. Angular, TypeScript, HTML, CSS, and JavaScript Security

### Authentication and authorization
- Treat the browser as an untrusted environment.
- Verify that backend APIs enforce all authorization independently.
- Check whether sensitive data or privileges are exposed through client-side state, routes, or bundles.
- Inspect token storage, token handling, session expiration, and logout behavior.
- Review route guards as navigation controls, not substitutes for server-side authorization.

### Cross-site scripting and unsafe rendering
- Check for unsafe HTML rendering, direct DOM manipulation, and bypasses of Angular's sanitization.
- Review use of innerHTML, direct DOM APIs, dynamic templates, and unsafe URL handling.
- Verify that user-controlled content is safely rendered.
- Inspect postMessage usage and validate message origins and payloads where relevant.
- Check for dangerous use of eval or dynamic code execution.

### HTTP, browser, and configuration
- Check API calls for accidental exposure of credentials or sensitive data.
- Review CORS configuration on the backend and understand that CORS is not an authorization mechanism.
- Inspect Content Security Policy and other relevant security headers where configuration is available.
- Check for secrets accidentally embedded in frontend environment files or build artifacts.
- Review redirects and externally supplied URLs for phishing or open-redirect risks.
- Inspect dependencies and unsafe package usage.

### Forms and frontend business logic
- Verify that frontend validation is backed by server-side validation.
- Check for sensitive values, privileged actions, or financial amounts that can be manipulated by a client.
- Review error messages for unnecessary internal information.
- Inspect caching and browser persistence of sensitive data.
- Confirm that UI permission checks do not replace API authorization.

## 5. SQL, Concurrency, and Financial Integrity

For payment-related systems, review:
- Duplicate transaction processing and idempotency.
- Race conditions during wallet creation, balance updates, or subscriber changes.
- Incorrect transaction isolation and missing atomicity.
- Validation of amount, currency, account, beneficiary, and transaction status.
- Trust in client-supplied transaction references or authorization results.
- Replay protection, request signing, timestamps, and nonce handling where applicable.
- Retry behavior after timeouts or ambiguous external-service responses.
- Duplicate notifications and inconsistent state transitions.
- Sensitive data in transaction logs, audit trails, and external request payloads.
- Integrity and access controls around reconciliation, refunds, reversals, and administrative actions.

Do not assume a distributed lock, in-memory limiter, or frontend duplicate-click prevention provides sufficient protection. Evaluate the actual deployment topology, failure modes, and persistent transaction guarantees.

Never infer that a financial transaction is safe