# dinks
As in "Double income, No kids"

Monorepo for the dinks app: a Go backend, a React web frontend, an Expo mobile
app, and the TypeScript package they share.

## Layout

| Path | What it is |
|---|---|
| `backend/` | Go API (MongoDB, JWT + OIDC auth). Dockerfile at `backend/Dockerfile`. |
| `web/` | React + Vite frontend served by nginx. Dockerfile at `web/Dockerfile`. |
| `mobile/` | Expo app, built to an Android APK. |
| `packages/shared/` | Types and date/cycle logic shared by web and mobile. |

## Local development

Requires Node 22+, Go 1.27+ and Docker.

```sh
docker compose up          # mongo + backend + web on :8080
```

Other useful commands, run from the repository root:

```sh
npm ci                     # install web, mobile and shared workspaces
npm run check              # typecheck every workspace
npm run test               # run workspace tests
npm run build:web          # build the web bundle
```

The data file format used by `/api/export` and `/api/import` is documented in
[IMPORT.md](./IMPORT.md).

## CI

Workflows live in [`.github/workflows`](./.github/workflows). The
`reusable-*.yaml` files are `workflow_call` templates; the rest are thin callers,
so behaviour stays identical to the home-cluster setup this repo was split out of.

| Workflow | Trigger | Does |
|---|---|---|
| `ci.yaml` | PR / push to `main` | Builds both images, hadolint, Trivy fs + image scan, dive layer analysis |
| `lint-backend.yaml` | PR / push to `main` | `golangci-lint` on `backend/` |
| `lint-web.yaml` | PR / push to `main` | TypeScript check across workspaces |
| `publish-web.yaml` | tag `dinks-web/v*` | Builds and pushes `ghcr.io/therayquaza/dinks/web` |
| `publish-backend.yaml` | tag `dinks-backend/v*` | Builds and pushes `ghcr.io/therayquaza/dinks/backend` |
| `publish-mobile.yaml` | tag `dinks-mobile/v*` | Builds the release APK and publishes a GitHub Release |
| `renovate.yml` | every 6h | Dependency updates |

Images are published to GHCR on tag push, with SLSA build provenance attached via
`actions/attest-build-provenance`.

## Releasing

```sh
git tag dinks-web/v0.2.5 && git push origin dinks-web/v0.2.5
git tag dinks-backend/v0.2.2 && git push origin dinks-backend/v0.2.2
```

After publishing, bump the matching image tag in the `home-cluster` repository
(`k8s/dinks/deployment*.yaml`), which ArgoCD syncs.

## Deployment

This repo owns the source and the images. The cluster manifests live in
[`TheRayquaza/home-cluster`](https://github.com/TheRayquaza/home-cluster) under
`k8s/dinks/`, applied by ArgoCD from `k8s/apps/dinks.yaml`.