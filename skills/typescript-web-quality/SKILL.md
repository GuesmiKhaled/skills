---
name: typescript-web-quality
description: Improve TypeScript, JavaScript, HTML, and CSS quality in web applications. Use for frontend debugging, type safety, browser behavior, accessibility, responsive design, and performance.
---

# Web Development Quality Standards

## TypeScript and JavaScript
- Follow the project's configured TypeScript and JavaScript versions.
- Prefer const by default and let when reassignment is necessary.
- Use meaningful names and small, focused functions.
- Prefer strict typing and avoid any where a safer type is available.
- Handle promises, asynchronous failures, and cancellation appropriately.
- Avoid unnecessary DOM manipulation when using Angular.
- Avoid eval, unsafe dynamic code execution, and unsafe HTML insertion.
- Keep business logic separate from presentation when practical.

## HTML
- Use semantic elements and correct document structure.
- Associate form controls with accessible labels.
- Provide meaningful alternative text for informative images.
- Preserve keyboard accessibility and visible focus indicators.
- Do not use clickable div elements when semantic interactive elements are appropriate.

## CSS
- Follow the existing design system and CSS conventions.
- Prefer reusable variables, scoped styles, and responsive layouts.
- Avoid duplicated declarations, excessive specificity, and unnecessary !important.
- Respect existing browser support requirements.
- Test important breakpoints and avoid unintended horizontal overflow.

## Verification
- Follow the repository's existing formatter, linter, and test configuration.
- Test edge cases, empty states, errors, and responsive behavior.
- Do not add new dependencies without justification and approval.
- Make the smallest safe change that meets the requirement.
