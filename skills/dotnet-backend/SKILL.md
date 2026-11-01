---
name: dotnet-backend
description: Implement, debug, review, optimize, and refactor C# and ASP.NET Core backend code, REST APIs, services, data access, authentication, and integrations. Use for all non-trivial .NET backend tasks.
---

# .NET Backend Engineering

## Before Changes
1. Inspect the solution structure, target frameworks, project references, and relevant source files.
2. Identify existing architectural patterns, dependency injection registrations, DTOs, interfaces, database access, and exception handling.
3. Determine the expected behavior and backward-compatibility requirements.
4. For non-trivial work, provide a concise plan and a file-by-file change list before editing.

## C# and ASP.NET Core
- Follow the C# language version and target framework already configured in the project.
- Use asynchronous APIs and async/await for I/O-bound operations where supported.
- Propagate CancellationToken through asynchronous call chains when appropriate.
- Use dependency injection and follow existing service lifetimes.
- Keep controllers or endpoints focused on HTTP concerns; place business logic in the appropriate existing application/service layer.
- Use DTOs for API contracts when consistent with the project architecture.
- Validate input at system boundaries and enforce authorization server-side.
- Use appropriate HTTP status codes and consistent error responses.
- Do not expose internal exception details or sensitive data in API responses.
- Avoid blocking asynchronous work with .Result or .Wait().
- Do not introduce unnecessary repositories, interfaces, generic abstractions, or new packages.

## Data Access
- Inspect the existing ORM, database provider, schema, and transaction patterns before making changes.
- Use parameterized queries for SQL and safe data-access APIs.
- Preserve transaction boundaries and existing business invariants.
- Check query performance, indexes, pagination, and unnecessary database round trips.
- Never modify production data or execute destructive database operations without explicit approval.

## Reliability and Observability
- Handle exceptions at the appropriate boundary; do not swallow exceptions.
- Use structured logging and meaningful event context.
- Never log credentials, access tokens, OTPs, payment card details, or sensitive customer information.
- Consider timeouts, cancellation, idempotency, and retry safety for external service integrations.

## Testing and Verification
- Follow the project's existing testing framework and test conventions.
- Test success cases, validation failures, authorization failures, exceptions, and important edge cases.
- Build the affected project or solution and run relevant tests when possible.
- Report commands executed and actual outcomes.
- Do not claim unexecuted tests passed.

## Constraints
- Do not silently change API contracts, database schemas, configuration keys, or business rules.
- Do not upgrade the target framework or packages without authorization.
- Prefer minimal, reviewable changes over rewriting entire files.
