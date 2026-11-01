---
name: angular-frontend
description: Build, debug, test, review, and refactor Angular applications using TypeScript, HTML templates, CSS, RxJS, forms, routing, and HTTP APIs.
---

# Angular Frontend Engineering

## Inspect First
- Inspect package.json, angular.json, tsconfig files, installed Angular version, and existing application structure.
- Follow the existing Angular version, component style, state-management strategy, and coding conventions.
- Reuse existing components, services, shared types, UI libraries, and utilities.
- Do not migrate a legacy application to standalone components, signals, or new APIs unless required and approved.

## Components and Architecture
- Keep components focused on presentation and user interaction.
- Move reusable business logic and HTTP communication into appropriate services or existing application layers.
- Prefer feature-oriented organization when consistent with the current project.
- Avoid oversized components, duplicated logic, and complex expressions in templates.
- Use dependency injection and lifecycle hooks appropriately.
- Clean up subscriptions and other resources according to the project's Angular and RxJS patterns.

## TypeScript
- Prefer strict types and explicit API models over any.
- Avoid unsafe type assertions and non-null assertions unless justified.
- Use interfaces or types that match the actual backend API contracts.
- Handle undefined, null, empty collections, and failed HTTP requests deliberately.
- Use RxJS operators appropriately and avoid unnecessary nested subscriptions.
- Avoid duplicate HTTP requests and memory leaks.

## HTML and Forms
- Use semantic HTML and accessible labels, keyboard interactions, and meaningful button text.
- Use the project's existing reactive or template-driven form approach.
- Apply appropriate validation and display understandable validation errors.
- Handle loading, empty, success, and error states.
- Never rely on frontend validation as the only security control.
- Escape or sanitize untrusted content appropriately; do not introduce unsafe DOM operations.

## CSS and Responsive UI
- Follow existing naming conventions, component encapsulation, design tokens, and styling libraries.
- Prefer maintainable, scoped styles over broad global selectors.
- Avoid unnecessary !important declarations and duplicated styles.
- Check responsive layouts, overflow, contrast, and keyboard focus.
- Do not introduce a new CSS framework or UI library without approval.

## Performance and Testing
- Avoid unnecessary change detection work, repeated expensive computations, and large unnecessary subscriptions.
- Follow the existing strategy for lazy loading, pagination, and data caching.
- Add or update tests using the project's existing testing tools.
- Run the relevant lint, test, and build commands when available.
- Report all verification results honestly.

## Constraints
- Preserve existing routes, API payloads, permissions, and business behavior unless explicitly requested.
- Do not upgrade Angular or replace major libraries without approval.
- Keep changes focused and explain any required backend contract changes.
