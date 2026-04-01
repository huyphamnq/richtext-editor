# Contributing

Thank you for contributing to Rich Text Editor.

This document explains how to set up the project, propose changes, and submit pull requests.

## 1. Development Setup

Prerequisites:

- Node.js 18+
- npm 9+

Install and run:

```bash
npm install
npm run dev
```

Build check:

```bash
npm run build
```

## 2. Branch Naming

Use short and descriptive branch names:

- feature/editor-signature
- fix/context-menu-hover
- docs/readme-bilingual

## 3. Commit Style

Use clear commit messages. Conventional Commit style is recommended:

- feat: add alignment for signature blocks
- fix: restore setImage command in custom image extension
- docs: add release notes for v1.0.0

## 4. Pull Request Guidelines

Before opening a PR:

- Rebase or merge latest main
- Confirm the app runs locally
- Run a production build successfully
- Update docs when behavior changes

PR description should include:

- Summary of changes
- Why the change is needed
- Manual test checklist
- Screenshots or short clips for UI changes

## 5. Code Style Expectations

- Keep changes focused and minimal
- Preserve existing project style and naming
- Avoid unrelated refactors in feature/fix PRs
- Add comments only where logic is non-obvious

## 6. Testing Guidance

Current project does not include an automated test suite.
Use a manual checklist for affected flows, for example:

- Text formatting commands
- Context menu and submenu hover behavior
- Image upload and alignment
- Signature drawing and insertion
- Export preview and download

## 7. Security and Sensitive Data

- Do not commit API keys or credentials
- Gemini API key is user-provided and stored in localStorage at runtime

## 8. Questions

If you are unsure about a change scope, open an issue first and describe:

- Problem statement
- Proposed approach
- Potential impact
