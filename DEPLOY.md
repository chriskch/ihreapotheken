# Deployment — ihreapotheken.htw-projekte.de (Coolify, Static build pack)

Target: `https://ihreapotheken.htw-projekte.de`
Build pack: **Static** (nginx, no Dockerfile, no build step)

This project is plain static HTML/JS/assets. Nothing is compiled — the
`ratgeber/` page renders client-side via `support.js`, which pulls React and
Babel from unpkg at runtime.

```
index.html                 full-page iframe of https://ihreapotheken.de/
ratgeber/index.html         standalone landing page (dc-runtime template)
ratgeber/support.js         generated dc-runtime (do not edit by hand)
ratgeber/uploads/, assets/  images and SVGs
robots.txt, llms.txt        production files for ihreapotheken.de (see note below)
nginx.conf                  reference web server config to paste into Coolify
```

## Prerequisites

### 1. DNS must be repointed — currently it is not

As of 2026-07-30 the hostname resolves to IONOS shared hosting, not a Coolify
server:

```
$ dig +short ihreapotheken.htw-projekte.de
217.160.0.67                     # IONOS Apache, placeholder page from 2021
```

Create/replace the DNS record so it points at the Coolify host:

```
ihreapotheken   A   <COOLIFY_SERVER_IP>    TTL 300
```

Let's Encrypt validates over HTTP against this record, so **SSL issuance will
fail until DNS has propagated.** Verify before deploying:

```
dig +short ihreapotheken.htw-projekte.de   # must return the Coolify server IP
```

### 2. Git repository

Coolify's Static build pack pulls from Git — it cannot take an upload. A local
repo already exists (`git log` in this directory). Push it to your host, e.g.:

```
git remote add origin git@github.com:<org>/ihreapotheken-preview.git
git push -u origin main
```

If the repo is private, connect Coolify to the host first via
**Settings → Sources** (GitHub App / deploy key).

## Coolify setup

1. **Project → New Resource → Public Repository** (or *Private Repository* via
   the configured source).
2. Repository URL: the remote from above. Branch: `main`.
3. **Build Pack: `Static`.**
4. **Base Directory: `/`** — the site root is the repo root.
5. **Domains:** `https://ihreapotheken.htw-projekte.de`
   Enter it with the `https://` scheme so Coolify requests a Let's Encrypt
   certificate and adds the HTTP→HTTPS redirect.
6. **Ports Exposes: `80`** (nginx default; usually prefilled).
7. **Web server configuration:** click **Generate** to produce the default
   nginx config for your Coolify version, then merge in the blocks marked
   `# ---- added:` from [`nginx.conf`](nginx.conf). They add gzip, asset
   caching, relative redirects, and the `X-Robots-Tag: noindex` header.
   If the generated default matches the shape of `nginx.conf`, paste it whole.
8. **Deploy.**

No environment variables, no health check path, no persistent storage needed.

## Verify after deploy

```bash
curl -sI https://ihreapotheken.htw-projekte.de/ | head -1          # 200
curl -sI https://ihreapotheken.htw-projekte.de/ratgeber | grep -i location   # /ratgeber/
curl -sI https://ihreapotheken.htw-projekte.de/ratgeber/ | grep -i x-robots  # noindex, nofollow
curl -s -o /dev/null -w '%{http_code}\n' \
  https://ihreapotheken.htw-projekte.de/ratgeber/uploads/IhreApotheken.de-Logo.png  # 200
```

Then open `/ratgeber/` in a browser and confirm the page actually renders — a
blank page means `support.js` could not reach unpkg or React failed to boot;
check the browser console for `[dc] failed to load React or boot`.

The above routing, headers, gzip and redirect behaviour were verified locally
against `nginx:alpine` with this `nginx.conf` before deployment.

## Notes and caveats

- **`robots.txt` is the production file for `ihreapotheken.de`.** It contains
  `Allow: /` for AI/search crawlers and a `Sitemap:` pointing at the production
  domain. Served from the staging host unchanged, it would invite indexing of a
  prototype that duplicates production content. The `X-Robots-Tag: noindex,
  nofollow` header in `nginx.conf` is what prevents that — the file is left
  untouched because it is itself a deliverable under review. If this host is
  ever meant to be public and indexed, remove the header deliberately.
- **`index.html` iframes the live site.** `https://ihreapotheken.de/` sends no
  `X-Frame-Options` or `frame-ancestors` (verified 2026-07-30), so the iframe
  loads. If production ever adds those headers, the root page goes blank —
  `/ratgeber/` is unaffected.
- **Runtime CDN dependency.** `support.js` loads React 18.3.1, ReactDOM and
  `@babel/standalone` from unpkg.com on every page view. The page will not
  render if unpkg is unreachable or blocked. Vendoring those three files into
  `ratgeber/assets/` would remove the dependency.
- **"Static" still runs a container.** Everything Coolify deploys is a
  container; the Static build pack means nginx is built and configured for you,
  with no Dockerfile or build command on our side.
- **Redeploys:** push to `main`. Enable *Auto Deploy* (webhook) in Coolify if
  you want pushes to publish automatically.
