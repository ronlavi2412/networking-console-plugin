# networking-console-plugin

OpenShift Console dynamic plugin for the Networking section (Services, Routes, Ingresses, NetworkPolicies, NADs, UDNs).

## Context

- [AGENTS.md](AGENTS.md) -- repo structure, key patterns, and review checklist
- [ARCHITECTURE.md](ARCHITECTURE.md) -- system design and data flow
- [CONTRIBUTING.md](CONTRIBUTING.md) -- coding standards and PR process

## Quick Reference

- **Stack**: TypeScript, React 18, PatternFly 6, webpack module federation, esbuild-loader
- **Path aliases**: `@utils/` = `src/utils/`, `@views/` = `src/views/`, `@styles/` = `src/styles/`
- **Lint**: `npm run lint` (ESLint + Prettier). `npm run lint-fix` to auto-fix.
- **Build**: `npm run build` (production), `npm run start` (dev server on port 9001)
- **Test**: `npm run test-cypress-headless` (Cypress E2E, requires `oc login` + `npm run start-console`)
- **i18n**: `npm run i18n` to regenerate locale files. Namespace: `plugin__networking-console-plugin`

## Key Rules

- Import order: React, packages, `@` aliases, relative, styles (enforced by `simple-import-sort`)
- Named exports sorted alphabetically (enforced by `perfectionist`)
- No `console.log` -- use `networkConsole.warn` for dev-only logging
- No hex colors in SCSS -- use PatternFly CSS variables
- Prefix CSS classes to avoid console style collisions
- Build resource URLs with `getResourceURL()` or `resourcePathFromModel()`, not string concatenation
- K8s data fetching via `useK8sWatchResource` only (no direct API calls)
- Prow OWNERS: `/lgtm` from reviewer + `/approve` from approver required to merge
