# Contributing

Thank you for contributing to **networking-console-plugin**. This document covers coding standards, the PR process, and testing requirements.

For initial setup (clone, install, run), see [README.md](README.md#quick-start).

## Coding Standards

### TypeScript

- Target: ES2016, JSX mode `react`, module resolution `node`.
- Strict unused-local checking is enabled (`noUnusedLocals: true`).
- Use path aliases (`@utils/`, `@views/`, `@styles/`) for imports instead of long relative paths.

### Import Ordering

Imports are sorted by `eslint-plugin-simple-import-sort` in this order:

1. Node.js builtins
2. React, then third-party packages
3. Internal `@`-prefixed path aliases
4. Side-effect imports
5. Parent relative imports (`../`)
6. Sibling relative imports (`./`)
7. Style imports (`.scss`, `.css`)

Run `npm run lint-fix` to auto-sort.

### Naming and Sorting

- Named exports and class members are sorted alphabetically by `eslint-plugin-perfectionist`.
- Components use PascalCase filenames matching the default export name (e.g., `ServiceRow.tsx` exports `ServiceRow`).
- Hooks use `use` prefix in camelCase (e.g., `useServiceColumn.ts`).
- Constants and utility files use camelCase (e.g., `constants.ts`, `utils.ts`).

### Styling

- Use SCSS, co-located with the component that owns the styles.
- Prefix CSS class names with the plugin name or a component-specific prefix to avoid conflicts with console styles.
- Do not use hex colors -- use [PatternFly CSS variables](https://www.patternfly.org/developer-resources/global-css-variables).
- Do not use naked element selectors (`table`, `div`) or `.pf-`/`.co-` prefixed classes.

### i18n

- All user-facing strings go through the `plugin__networking-console-plugin` i18n namespace.
- Use the `useNetworkingTranslation()` hook for simple translations.
- Use the `Trans` component from `react-i18next` when strings contain JSX interpolation.
- Use the standalone `t()` function from `@utils/hooks/useNetworkingTranslation` for translations outside React components.
- Run `npm run i18n` after adding or changing translatable strings. Commit the updated locale files.

## Project Structure Conventions

### View Directories

Each networking resource lives in `src/views/{resource}/` with a consistent internal structure:

```text
src/views/{resource}/
  manifest.ts          # Console SDK extension declarations and exposed modules
  list/                # List page
    {Resource}List.tsx
    components/        # Row components, empty states
    hooks/             # useXxxColumn, useXxxFilters
    utils/             # Column definitions, sorting
  details/             # Details page
    {Resource}DetailsPage.tsx
    components/        # Title, section components
    hooks/             # useXxxTabs, useXxxActions
    tabs/              # Tab content (details/, yaml/, metrics/, pods/)
  actions/             # Resource action menus
    hooks/             # useXxxActions hook
  form/                # Create/edit forms (not all resources have forms)
    components/
    hooks/
    utils/
```

### Shared Utilities

Reusable components, hooks, models, and helpers live in `src/utils/`:

- `components/` -- shared UI components (modals, dropdowns, loading states, selectors)
- `hooks/` -- shared hooks (`useNetworkingTranslation`, `useIsAdmin`, `usePagination`)
- `models/` -- Kubernetes resource model definitions (`K8sModel` objects)
- `resources/` -- resource accessor helpers (`getName`, `getNamespace`, `getResourceURL`)
- `constants/` -- shared constants, documentation URLs, UI path constants
- `flags/` -- feature flag detection (networking dynamic flag, UDN enabled flag)
- `telemetry/` -- Segment analytics event logging

### Plugin Manifest

`plugin-manifest.ts` at the repo root aggregates all view manifests. Each view module exports:
- `{Resource}Extensions` -- array of `EncodedExtension` objects declaring console extension points
- `{Resource}ExposedModules` -- mapping of webpack module federation entry points

## PR Process

### Branch Strategy

The project maintains multiple active release branches (`release-4.16` through `release-4.23` plus `main`). Fixes typically land on `main` first and are cherry-picked to affected release branches.

PR titles for cherry-picks follow the format: `[release-X.YY] OCPBUGS-XXXXX: description`.

### Code Review

The repo uses Prow with an `OWNERS` file. Merging requires:

1. **`/lgtm`** from at least one reviewer listed in `OWNERS` `reviewers:`
2. **`/approve`** from at least one approver listed in `OWNERS` `approvers:`
3. All CI checks passing

Reviewers and approvers are distinct roles -- a reviewer can `/lgtm` but only an approver can `/approve`. Both labels are required for merge.

### Review Expectations

- Justify every change in the PR. Reviewers question unnecessary modifications (unused import removals, unrelated formatting changes).
- Keep PRs focused on a single concern. CVE remediation, dependency upgrades, and feature work belong in separate PRs.
- Verify i18n locale files are updated if translatable strings changed.
- CodeRabbit provides automated review feedback. Address CodeRabbit comments or explain why they do not apply.

### Commit Conventions

Commit messages reference Jira tickets when applicable using the `OCPBUGS-XXXXX:` prefix. Cherry-pick commits include the target branch in the message.

## Testing

### Cypress E2E Tests

Integration tests live in `integration-tests/` and use Cypress:

```bash
npm run test-cypress            # Interactive Cypress runner
npm run test-cypress-headless   # Headless mode (CI)
```

Tests require an authenticated OpenShift cluster (`oc login`) and the console running (`npm run start-console`).

### CI (Prow)

The `test-prow-e2e.sh` script runs in the OpenShift CI Prow environment. It reads cluster credentials from environment variables, installs dependencies with `npm ci`, and runs Cypress headlessly. Test artifacts (screenshots) are copied to `$ARTIFACT_DIR`.

### Reporting

Generate a combined Cypress report:

```bash
npm run cypress-merge
npm run cypress-generate
```

The HTML report appears in `integration-tests/screenshots/`.

## Linting

```bash
npm run lint       # Check for issues
npm run lint-fix   # Auto-fix
```

ESLint runs on both `src/` and `integration-tests/`. See [README.md](README.md#linting) for the full list of enforced rules.
