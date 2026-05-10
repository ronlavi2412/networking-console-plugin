# Architecture

## System Context

networking-console-plugin is an OpenShift Console dynamic plugin that replaces the built-in networking UI. It runs as a standalone nginx container serving webpack-bundled JavaScript. The OpenShift Console loads the plugin at runtime via [webpack module federation](https://webpack.js.org/concepts/module-federation/) and the `ConsolePlugin` custom resource.

```text
+-----------------+       module federation       +----------------------------+
|  OpenShift      | <---------------------------> |  networking-console-plugin  |
|  Console        |   (loads JS chunks at runtime) |  (nginx on port 80)        |
|  (browser SPA)  |                                +----------------------------+
+--------+--------+                                       |
         |                                                 |
         v                                                 v
+--------+--------+                              +--------+--------+
| Kubernetes API  |                              | Build artifacts  |
| Server          |                              | in dist/         |
+-----------------+                              +-----------------+
```

The plugin communicates with the Kubernetes API server exclusively through the Console SDK's data-fetching hooks (`useK8sWatchResource`). It does not make direct API calls.

## Component Architecture

### Plugin Entry Point

`plugin-manifest.ts` at the repo root defines all console extension points and webpack module federation entry points. It aggregates manifests from each view module.

### View Modules

Each networking resource has its own directory under `src/views/` with a consistent structure:

| Directory | Resource | Kubernetes API |
|---|---|---|
| `services/` | Service | `core/v1` |
| `routes/` | Route | `route.openshift.io/v1` |
| `ingresses/` | Ingress | `networking.k8s.io/v1` |
| `networkpolicies/` | NetworkPolicy | `networking.k8s.io/v1` |
| `nads/` | NetworkAttachmentDefinition | `k8s.cni.cncf.io/v1` |
| `udns/` | UserDefinedNetwork, ClusterUserDefinedNetwork | `k8s.ovn.org/v1` |
| `createprojectmodal/` | Project (creation modal) | `project.openshift.io/v1` |

Each view module exports a `manifest.ts` that declares:
- **Extension types**: `console.page/resource/list`, `console.page/resource/details`, `console.navigation/resource-ns`, `console.yaml-template`, `console.resource/details-item`, `console.page/route`
- **Exposed modules**: Webpack module federation entry points mapping to React components

### Data Flow

```text
Kubernetes API
      |
      v
useK8sWatchResource (Console SDK)
      |
      v
useListPageFilter (filtering/search)
      |
      v
VirtualizedTable / HorizontalNav
      |
      v
Row / Tab components (rendering)
```

1. **Watch**: `useK8sWatchResource` establishes a WebSocket watch on the Kubernetes API and returns `[data, loaded, error]`.
2. **Filter**: `useListPageFilter` provides client-side filtering, returning `[allData, filteredData, onFilterChange]`.
3. **Display**: `VirtualizedTable` renders filtered data with column definitions from `use{Resource}Column` hooks. Detail pages use `HorizontalNav` with tab arrays from `use{Resource}Tabs` hooks.
4. **Actions**: Resource action menus are built with `use{Resource}Actions` hooks returning action arrays for dropdowns.

### Shared Utilities

`src/utils/` provides cross-cutting concerns:

- **`components/`** -- ~40 reusable UI components. Each component gets its own directory with a PascalCase name matching the export (e.g., `BaseModal/BaseModal.tsx`, `StatusBox/StatusBox.tsx`).
- **`hooks/`** -- Shared React hooks for translation, admin detection, pagination, namespace tracking.
- **`models/`** -- `K8sModel` definitions for CRDs not provided by the Console SDK (MultiNetworkPolicy, UserDefinedNetwork, ClusterUserDefinedNetwork, NodeNetworkConfigurationPolicy, NodeNetworkState, EndpointSlice).
- **`resources/`** -- Accessor functions for K8s resources (`getName`, `getNamespace`, `getLabels`, `getAnnotations`, `getResourceURL`, `resourcePathFromModel`). Subdirectories hold resource-specific utilities (NADs, NetworkPolicies, UDNs, VMs).
- **`flags/`** -- Feature flag registration and detection for conditional UI rendering.
- **`telemetry/`** -- Segment analytics integration for tracking resource creation events.

## Dependencies

### Console SDK

The plugin depends on three Console SDK packages (currently aligned to the 4.22 prerelease):
- `@openshift-console/dynamic-plugin-sdk` -- runtime APIs (hooks, components, types)
- `@openshift-console/dynamic-plugin-sdk-internal` -- internal console APIs (telemetry)
- `@openshift-console/dynamic-plugin-sdk-webpack` -- webpack plugin for module federation

SDK versions correspond to OpenShift release branches. When a new release branch is cut, the SDK packages are bumped to match.

### UI Framework

- **PatternFly 6** (`@patternfly/react-core`, `@patternfly/react-table`, `@patternfly/react-icons`, `@patternfly/react-tokens`, `@patternfly/react-charts`, `@patternfly/react-component-groups`)
- **Victory Charts** (area, bar, line, pie, scatter, and associated chart primitives)
- **react-hook-form** for form state management
- **react-i18next** for internationalization
- **react-router** (v7) for client-side routing

### Kubernetes Types

`@kubevirt-ui/kubevirt-api` provides TypeScript types for Kubernetes resources and the `modelToGroupVersionKind`/`modelToRef` utility functions.

## Build Pipeline

### Local Development

```text
webpack dev server (port 9001)
  + esbuild-loader (TypeScript transpilation)
  + ConsoleRemotePlugin (module federation)
  + CopyWebpackPlugin (locales -> dist)
  + TsconfigPathsPlugin (path alias resolution)
      |
      v
  Hot-reloading via WebSocket
```

### Production Build

`npm run build` sets `NODE_ENV=production`, enables minification, and produces chunked output with content hashes in `dist/`.

### Container Build

The `Dockerfile` uses a multi-stage build:
1. **Builder stage**: `registry.ci.openshift.org/ocp/builder:rhel-9-base-nodejs-openshift-4.22` runs `npm ci && npm run build`
2. **Runtime stage**: `registry.ci.openshift.org/ocp/4.22:base-rhel9` installs nginx and copies `dist/` to serve static files

`Dockerfile.art` is the ART (Automated Release Tooling) variant used by the OpenShift build system.

### CI

The project uses OpenShift CI (Prow). The `test-prow-e2e.sh` script runs Cypress E2E tests against a live cluster in the CI environment. The `OWNERS` file controls Prow-based code review automation (`/lgtm` and `/approve` labels).

## Release Strategy

The project maintains multiple active release branches (`release-4.16` through `release-4.23`) alongside `main`. Each release branch tracks a specific OpenShift version with matching Console SDK and builder image versions.

Fixes land on `main` first and are cherry-picked to affected release branches. Cherry-pick PRs follow the naming convention `[release-X.YY] OCPBUGS-XXXXX: description`.

## Deployment

The Helm chart in `charts/openshift-console-plugin/` deploys the plugin to an OpenShift cluster. It creates the `ConsolePlugin` CR, a Deployment running the nginx container, a Service, and optionally a NetworkPolicy. The console operator configuration must enable the plugin for it to appear in the UI.
