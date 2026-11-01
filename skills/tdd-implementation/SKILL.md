---
name: tdd-implementation
description: Implement features, bug fixes, and refactoring using test-driven development (TDD) in .NET Core, C#, ASP.NET Core, Angular, TypeScript, JavaScript, and web applications. Use when correctness, regression prevention, and testable implementation are important.
---

# Test-Driven Development (TDD)

## 1. Objective

Implement reliable, maintainable code through a disciplined Red-Green-Refactor workflow.

- Red: Write a test that expresses the expected behavior and verify that it fails for the intended reason.
- Green: Implement the smallest production-code change needed to make the test pass.
- Refactor: Improve the implementation without changing observable behavior, keeping all relevant tests passing.

Never claim that a test passed, failed, or was executed unless the corresponding command was actually run and its result was inspected.

## 2. Inspect Before Coding

Before modifying files:

1. Inspect the repository structure, solution or workspace configuration, target framework, dependency versions, and existing test projects.
2. Read the implementation, related interfaces, DTOs, callers, configuration, and existing tests.
3. Identify existing naming conventions, architectural boundaries, test frameworks, mocks, fixtures, and test-data builders.
4. Identify acceptance criteria, edge cases, backward-compatibility constraints, and possible security implications.
5. Determine the smallest appropriate testing scope.

Do not assume a testing framework or introduce a new package before checking the existing project.

## 3. Plan First

For non-trivial tasks, present a concise implementation plan before editing.

Include:
- Expected behavior and acceptance criteria.
- Files to create or modify and their purpose.
- Tests required and important edge cases.
- External dependencies, database interactions, API contracts, and risks.
- Verification commands appropriate for the repository.

Prefer a focused file manifest over broad speculative changes.

Do not proceed with destructive, production, irreversible, or explicitly approval-gated operations without authorization.

For simple, low-risk fixes, use a proportionate plan and avoid unnecessary ceremony.

## 4. Define Acceptance Criteria

Translate the requirement into observable behavior before writing code.

Cover, where relevant:
- Normal successful execution.
- Invalid, missing, null, or boundary inputs.
- Authorization and validation failures.
- Exceptions, dependency failures, timeouts, and cancellation.
- Empty results, duplicate requests, and concurrent operations.
- Regression cases for the original defect.
- Compatibility with existing API contracts and business rules.

Use concrete inputs and expected outcomes. Avoid tests that merely reproduce the current implementation instead of verifying the requirement.

## 5. RED — Write a Failing Test

1. Add or update the most focused relevant test.
2. Use the repository's established testing framework and conventions.
3. Run the test before implementing the fix.
4. Confirm that it fails because the expected behavior is missing or incorrect.
5. Inspect the failure message to rule out compilation errors, incorrect setup, missing dependencies, and unrelated failures.

Do not deliberately introduce broken production code merely to demonstrate a failure.

If a failing test cannot be executed because of environment limitations, report that limitation and continue only with an explicit distinction between verified and unverified work.

## 6. GREEN — Implement the Smallest Change

1. Modify production code only as needed to satisfy the failing test.
2. Preserve existing architecture, public contracts, database behavior, and business rules unless the task explicitly requires a change.
3. Reuse established services, abstractions, validation, error-handling, and data-access patterns.
4. Avoid unrelated refactoring, speculative