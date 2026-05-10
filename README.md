# networking-console-plugin

## About

OpenShift Console dynamic plugin for the **Networking** section. Provides list, detail, form, and YAML views for core Kubernetes and OpenShift networking resources: Services, Routes, Ingresses, NetworkPolicies, NetworkAttachmentDefinitions (NADs), and UserDefinedNetworks (UDNs/CUDNs).

The plugin runs as a sidecar served by nginx and registered via the `ConsolePlugin` custom resource. It uses [webpack module federation](https://webpack.js.org/concepts/module-federation/) through the `@openshift-console/dynamic-plugin-sdk` to load into the running console at runtime.

## Prerequisites

- [Node.js](https://nodejs.org/) (version compatible with the CI builder image)
- [npm](https://www.npmjs.com/)
- [oc](https://console.redhat.com/openshift/downloads) CLI authenticated to an OpenShift cluster
- [Docker](https://www.docker.com) or [podman 3.2.0+](https://podman.io) (for running the console container)

## Quick Start

```bash
git clone https://github.com/openshift/networking-console-plugin.git
cd networking-console-plugin
npm ci
```

In one terminal, start the plugin dev server:

```bash
npm run start
```

In another terminal, log into your cluster and start the console:

```bash
oc login https://api.your-cluster.example.com:6443
npm run start-console
```

Navigate to `http://localhost:9000`. The plugin serves on port 9001 with CORS enabled.

### Running with Companion Plugins

The `start-console.sh` script accepts additional plugin names as arguments. It clones, installs, and starts each companion plugin on sequential ports (9002, 9003, ...) and registers them with the console container:

```bash
npm run start-console -- monitoring-plugin nmstate-console-plugin
```

Supported companion plugins: `monitoring-plugin`, `nmstate-console-plugin`, `kubevirt-plugin`.

### Apple Silicon + Podman

If `npm run start-console` fails on a Mac with Apple silicon, install `qemu-user-static` in the podman VM:

```bash
podman machine ssh
sudo -i
rpm-ostree install qemu-user-static
systemctl reboot
```

### VS Code Remote Container

1. Create `.devcontainer/dev.env` with cluster credentials (`OC_PLUGIN_NAME`, `OC_URL`, `OC_USER`, `OC_PASS`).
2. Open the folder in a Remote Container (`Ctrl+Shift+P` > "Remote Containers: Open Folder in Container...").
3. Run `npm run start` and navigate to `http://localhost:9000`.

## Development

### Scripts

| Command | Description |
|---|---|
| `npm run start` | Start webpack dev server on port 9001 |
| `npm run start-console` | Start OpenShift Console container connected to the cluster |
| `npm run build` | Production build to `dist/` |
| `npm run build-dev` | Development build to `dist/` |
| `npm run lint` | Run ESLint on `src/` and `integration-tests/` |
| `npm run lint-fix` | Auto-fix ESLint issues |
| `npm run i18n` | Extract and update i18n translation strings |
| `npm run test-cypress` | Open Cypress test runner (interactive) |
| `npm run test-cypress-headless` | Run Cypress tests headlessly |

### Path Aliases

The project uses TypeScript path aliases defined in `tsconfig.json`:

| Alias | Path |
|---|---|
| `@utils/*` | `src/utils/*` |
| `@views/*` | `src/views/*` |
| `@styles/*` | `src/styles/*` |

### Linting

ESLint enforces:
- Alphabetical sorting of named exports and class members via `eslint-plugin-perfectionist`
- Import grouping and ordering via `eslint-plugin-simple-import-sort` (React first, packages, internal `@` imports, relative imports, styles last)
- No `console.log` (`no-console: error`)
- No nested ternaries
- No unused variables (except those prefixed with `_`)
- Prettier formatting (single quotes, trailing commas, 100-char print width, 2-space indent)

Stylelint rules (when present) disallow hex colors (use PatternFly CSS variables) and naked element selectors or `.pf-`/`.co-` prefixed classes to prevent overwriting console styles.

### i18n

All user-facing strings use the `plugin__networking-console-plugin` namespace. Two patterns are in use:

```tsx
// Hook-based (most components)
const { t } = useNetworkingTranslation();
return <span>{t('My label')}</span>;
```

```tsx
// Trans component (JSX interpolation in strings)
import { Trans } from 'react-i18next';
return <Trans t={t} ns="plugin__networking-console-plugin">See <a>docs</a></Trans>;
```

Run `npm run i18n` to regenerate locale files in `locales/`. Supported locales: en, es, fr, ja, ko, zh.

## Docker Image

```bash
docker build -t quay.io/my-repository/networking-console-plugin:latest .
docker push quay.io/my-repository/networking-console-plugin:latest
```

On Apple silicon, add `--platform=linux/amd64`.

## Deployment

A [Helm](https://helm.sh) chart in `charts/openshift-console-plugin/` deploys the plugin to an OpenShift cluster:

```bash
helm upgrade -i networking-console-plugin charts/openshift-console-plugin \
  -n plugin__networking-console-plugin --create-namespace \
  --set plugin.image=quay.io/my-repository/networking-console-plugin:latest
```

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for coding standards, PR process, and review guidelines.

## References

- [Console Plugin SDK](https://github.com/openshift/console/tree/master/frontend/packages/console-dynamic-plugin-sdk)
- [Dynamic Plugin Enhancement Proposal](https://github.com/openshift/enhancements/blob/master/enhancements/console/dynamic-plugins.md)
- [PatternFly](https://www.patternfly.org/)
