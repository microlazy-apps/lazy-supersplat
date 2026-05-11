# lazy-supersplat maintainer notes

懒猫微服 (LazyCat MicroServer) lpk wrapper for
[playcanvas/supersplat](https://github.com/playcanvas/supersplat) —
PlayCanvas's open-source, browser-native 3D Gaussian Splat editor.

Upstream is vendored as a git subtree at `vendor/supersplat/`.
**All adaptations live in `patches/`** — `vendor/supersplat` stays
pristine. Build/publish/bootstrap logic is **not** in this repo —
it lives in
[microlazy-apps/lazycat-ci](https://github.com/microlazy-apps/lazycat-ci).

## Lazycat appstore identifiers

- **package id**: `cloud.lazycat.app.supersplat`
- **app_id**: TBD (recorded after first bootstrap)
- **subdomain**: `supersplat` → `https://supersplat.<box-domain>`
- **bootstrap workflow**: when re-running `bootstrap-app.yml` to
  resubmit a fix, pass `app_id=<NUM>` so the workflow skips
  `/app/create` (which would 500 on duplicate package).

## Architecture: single nginx serving a static SPA

SuperSplat is a pure browser app — rollup compiles TypeScript to a
static `dist/` directory and there is **no backend**. All editing
happens client-side on WebGL via PlayCanvas; nothing reaches the
server beyond serving the bundle.

```
  browser ───► nginx :80 ───► /usr/share/nginx/html (dist/)
```

The patch adds three files into `vendor/supersplat/`:

- `Dockerfile` — two-stage:
  1. `node:22-alpine` → `npm ci` → `npm run build` → `dist/`
  2. `nginx:1.27-alpine` → copies `dist/` to `/usr/share/nginx/html`
- `.dockerignore` — drops node_modules, dist, .git, docs.
- `nginx.conf` — single `server` block:
  - Health probe at `/health`
  - `/sw.js` and `/index.html` set `Cache-Control: no-store` so
    service-worker updates and HTML edits land immediately on the
    next release.
  - Hashed bundle assets + `/static/*` get `immutable; max-age=1y`.
  - SPA fallback `try_files $uri $uri/ /index.html`.

The build uses `BASE_HREF=""` so emitted asset paths are relative to
`/`. This is what nginx serves at the lazycat subdomain root.

## Why this is "vendor mode" instead of `FROM ghcr.io/...`

Upstream doesn't publish a Docker image — only a hosted instance at
`https://superspl.at/editor`. We have to build from source. The
upstream UI itself does not hard-code the superspl.at URL anywhere
that's user-visible, so no patch beyond the Dockerfile is needed.

## No deploy params (intentional)

There is nothing for the user to configure server-side. The Web UI
exposes its own settings (export targets, PlayCanvas publish creds)
in-browser and persists them to localStorage — not to the lazycat
filesystem. If you ever need install-time params (e.g. for an
upstream OIDC-protected publishing endpoint), add
`lazycat/lzc-deploy-params.yml` and the `deploy-params:` line in
`lzc-build.yml`.

## No persistent volumes (intentional)

All state is in the browser. Nothing under `/lzcapp/var/...` needs
to be bind-mounted.

## Patches workflow

```sh
# Apply patches into vendor for live editing:
git apply patches/01-lazycat-dockerfile.patch -p1 --directory=vendor/supersplat

# Make changes...
vim vendor/supersplat/nginx.conf

# Stage new files so git diff sees them:
git add -N vendor/supersplat/<any-new-files>

# Regenerate the patch:
git diff --no-color --relative=vendor/supersplat \
  vendor/supersplat/ > patches/01-lazycat-dockerfile.patch

# Restore vendor for clean commit (only patches/* should be tracked):
rm vendor/supersplat/Dockerfile vendor/supersplat/.dockerignore vendor/supersplat/nginx.conf
git reset -q HEAD vendor/supersplat/Dockerfile vendor/supersplat/.dockerignore vendor/supersplat/nginx.conf

# Verify:
git apply --check patches/01-lazycat-dockerfile.patch \
  -p1 --directory=vendor/supersplat
```

CI applies patches in lexical order inside lazycat-ci's docker job
before `docker build`, with `--directory=vendor/supersplat`.

## Lazycat OIDC integration (stage 2 — not yet wired)

Stage 2 is "对接 OIDC". Since SuperSplat has no native auth layer,
we will need a thin auth proxy in front of nginx that:

- Accepts only authenticated requests (via the lazycat platform's
  built-in OIDC at `/sys/oauth/auth`)
- Sets `application.oidc_redirect_path: /oauth2/callback` in the
  manifest
- Reverse-proxies everything else to the existing nginx :80

Likely implementation:
- Drop in `oauth2-proxy` as a sibling process under `supervisord`
  (or as a separate service) listening on :8080, pointed at nginx
  :80 as upstream. Then point lazycat's `upstreams.location: /` at
  `http://main:8080` instead of `:80`.
- The six `LAZYCAT_AUTH_OIDC_*` env vars get mapped into oauth2-proxy
  config (`--provider=oidc --oidc-issuer-url=...`,
  `--client-id=$LAZYCAT_AUTH_OIDC_CLIENT_ID`,
  `--redirect-url=https://${LAZYCAT_APP_DOMAIN}/oauth2/callback`,
  `--cookie-secret=$(stable_secret OAUTH2_COOKIE_SECRET)`).

This change keeps `vendor/supersplat` untouched — only the patched
Dockerfile and manifest gain a couple of extra files / env vars.

## Updating SuperSplat upstream

```sh
git subtree pull --prefix=vendor/supersplat \
  https://github.com/playcanvas/supersplat.git main --squash
git apply --check patches/01-lazycat-dockerfile.patch \
  -p1 --directory=vendor/supersplat
```

Upstream is active (renovate-managed deps). When the patch fails to
apply, the usual reason is one of:
- A new file at the root of the repo with the same name as one of
  our new files — unlikely (Dockerfile / .dockerignore / nginx.conf).
- The patches workflow above lets you re-derive cleanly.

## Local lpk smoke test

Requires `lzc-cli`, `envsubst` (gettext), and Docker on PATH.

```sh
# one-time
git clone https://github.com/microlazy-apps/lazycat-ci ~/lazycat-ci

# in this repo:
bash ~/lazycat-ci/scripts/build-lpk.sh \
  --image lazy-supersplat:dev \
  --version 0.0.0-dev \
  --docker-context ./vendor/supersplat \
  --patches-target vendor/supersplat
# → lazycat/cloud.lazycat.app.supersplat-0.0.0-dev.lpk
```

Build is moderate — the `npm ci` step downloads ~600MB of node_modules
(playcanvas engine + PCUI + rollup chain). Subsequent rebuilds are
cached. Expect ~3-5 min on a fresh build.

## Versioning notes

- Upstream supersplat uses its own `2.x.y` line (currently 2.25.1).
- This wrapper picks its own semver, starting at `v0.1.0`. Stage 1
  (no OIDC) lands at `v0.1.x`; stage 2 (OIDC wired) at `v0.2.0`.
- Lazycat appstore enforces strict monotonic semver per package id.
