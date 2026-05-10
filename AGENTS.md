# Agents Guide

## Overview

AI-specific guidance for working with the networking-console-plugin codebase.

For coding standards, PR process, and testing, see [CONTRIBUTING.md](CONTRIBUTING.md).

## Repo Map

```text
networking-console-plugin/
  plugin-manifest.ts           # Root manifest aggregating all view extensions
  webpack.config.ts            # Webpack config with ConsoleRemotePlugin
  tsconfig.json                # TypeScript config with path aliases
  package.json                 # npm scripts, dependencies, consolePlugin metadata
  OWNERS                       # Prow reviewers and approvers
  Dockerfile / Dockerfile.art  # Multi-stage build (Node build -> nginx serve)
  charts/                      # Helm chart for cluster deployment
  locales/{en,es,fr,ja,ko,zh}/ # i18n translation JSON files
  integration-tests/           # Cypress E2E tests
  src/
    views/                     # Feature directories (one per networking resource)
      services/                # Kubernetes Services
      routes/                  # OpenShift Routes
      ingresses/               # Kubernetes Ingresses
      networkpolicies/         # Kubernetes NetworkPolicies
      nads/                    # NetworkAttachmentDefinitions (k8s.cni.cncf.io)
      udns/                    # UserDefinedNetworks and ClusterUserDefinedNetworks (k8s.ovn.org)
      createprojectmodal/      # Project creation with UDN support
    utils/
      components/              # ~40 shared UI components
      hooks/                   # Shared React hooks
      models/                  # K8sModel definitions for CRDs
      resources/               # Resource accessor utilities (getName, getResourceURL)
      constants/               # Shared constants and documentation URLs
      flags/                   # Feature flag detection
      telemetry/               # Segment analytics event tracking
    templates/                 # YAML templates for resource creation
    styles/                    # Global SCSS
```

## Key Patterns

### Extension Registration

Each view module provides a `manifest.ts` that exports `{Resource}Extensions` and `{Resource}ExposedModules`. The root `plugin-manifest.ts` imports and spreads these into the unified plugin metadata. When adding a new resource view:

1. Create `src/views/{resource}/manifest.ts` with extension declarations
2. Import and spread into `plugin-manifest.ts` `extensions` and `pluginMetadata.exposedModules`

### K8s Resource Models

Resource models are defined as `K8sModel` objects in `src/utils/models/`. Each model specifies `apiGroup`, `apiVersion`, `kind`, `plural`, and whether it is namespaced. CRD models include i18n `labelKey`/`labelPluralKey` comments for extraction.

Group-version-kind tuples are created with `modelToGroupVersionKind()` and refs with `modelToRef()` from `@kubevirt-ui/kubevirt-api/console`.

### Console SDK Integration

The plugin uses `@openshift-console/dynamic-plugin-sdk` for:
- **Data fetching**: `useK8sWatchResource` for single resources or lists
- **List pages**: `ListPageHeader`, `ListPageBody`, `ListPageFilter`, `VirtualizedTable`
- **Detail pages**: `HorizontalNav` with tab-based layout
- **Access control**: `createAccessReview` for button visibility
- **Navigation**: `useActiveNamespace`, React Router `useNavigate`

### URL Construction

Resource URLs follow the pattern `/k8s/{namespace-scope}/{group}~{version}~{kind}/{name}`. Use `getResourceURL()` or `resourcePathFromModel()` from `src/utils/resources/shared.ts` rather than building URLs manually.

### Form Patterns

Resources with create/edit forms use a `SyncedEditor` component that provides a toggle between a structured form view and a YAML editor. Forms use `react-hook-form` for state management.

### i18n

Two translation patterns are in use:
- `useNetworkingTranslation()` hook returning `{ t }` -- used in most components
- `Trans` component from `react-i18next` -- used when strings contain JSX elements (links, formatted text)
- Standalone `t()` function from `@utils/hooks/useNetworkingTranslation` -- used outside React render context

### Telemetry

Resource creation events are logged via `logNetworkingEvent()` in `src/utils/telemetry/`. Events include `NAD_CREATED`, `UDN_CREATED`, `CUDN_CREATED` with metadata about the resource configuration.

### Feature Flags

The plugin registers feature flags via console extensions:
- `enableNetworkingDynamicFlag` -- sets a flag when the plugin loads
- `useUDNEnabledFlag` -- hook-based flag for UDN support detection
- `NMSTATE_PLUGIN_ENABLED` -- model-based flag checking for NodeNetworkState CRD

## Review Checklist

When reviewing changes to this codebase:

- [ ] New components follow the `src/views/{resource}/{list|details|form|actions}/` directory structure
- [ ] Manifest files are updated in both the view's `manifest.ts` and `plugin-manifest.ts`
- [ ] All user-facing strings use `useNetworkingTranslation()` or `Trans` with the correct namespace
- [ ] `npm run i18n` was run and locale file changes are committed
- [ ] CSS classes are prefixed to avoid collisions with console styles
- [ ] No hex colors in SCSS (use PatternFly CSS variables)
- [ ] Import order follows the configured grouping (React, packages, `@` aliases, relative, styles)
- [ ] Named exports are alphabetically sorted (perfectionist plugin)
- [ ] No `console.log` statements (use `networkConsole.warn` from `@utils/utils/helpers` for dev-only logging)
- [ ] New K8sModel definitions include `labelKey` and `labelPluralKey` with i18n extraction comments
- [ ] Resource URLs are built with `getResourceURL()` or `resourcePathFromModel()`, not string concatenation
