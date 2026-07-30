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

### 2. Git repository — done

Coolify's Static build pack pulls from Git — it cannot take an upload.
`origin` is `git@github.com:chriskch/ihreapotheken.git`, branch `main` is
pushed. Redeploys are `git push` + *Redeploy* in Coolify.

If the repo is private, Coolify must be connected to GitHub via
**Settings → Sources** (GitHub App / deploy key) — since the app deploys, this
is already in place.

## Coolify setup

Application: **GEO Lab → production → IHRE APOTHEKEN** (`hosting-europe-3`).

Already configured:

| Field | Value |
| --- | --- |
| Build Pack | `Static` |
| Static Image | `nginx:alpine` |
| Base Directory | `/` |
| Domains | `https://ihreapotheken.htw-projekte.de` |
| Ports Exposes | `80` |

Still open:

1. **Custom Nginx Configuration** — paste the contents of
   [`nginx.conf`](nginx.conf). It is Coolify's generated default plus the
   changes marked `# ADDED` / `# CHANGED`: gzip, asset caching, relative
   redirects behind the TLS proxy, `server_tokens off`, the
   `X-Robots-Tag: noindex` header, and two fixes to the generated default
   (see below).

   The field is written to `/etc/nginx/conf.d/default.conf`, which the base
   image includes from **inside** the `http {}` block. It must contain a bare
   `server { }` block — an `events {}` or `http {}` block there makes nginx
   refuse to start (`"events" directive is not allowed here`) and the container
   crash-loops.

   `nginx.conf` is kept comment-free so it can be pasted as-is. What it adds on
   top of the generated default, and why:
   - `absolute_redirect off` — Coolify terminates TLS at its proxy and forwards
     plain HTTP. Without this, nginx's trailing-slash redirect sends an absolute
     `http://...` Location and drops the client off HTTPS for a hop.
   - `add_header X-Robots-Tag "noindex, nofollow" always` — see the robots.txt
     note under *Notes and caveats*. It is repeated inside both `location`
     blocks because nginx's `add_header` is not additive: a location that sets
     any `add_header` stops inheriting the server-level ones.
   - `server_tokens off` — do not advertise the nginx patch version.
   - `gzip` for text types, `Cache-Control` for images/fonts (30d) and
     `no-cache, must-revalidate` for HTML/JS/TXT so content edits go live on the
     next request.

   Two defects in Coolify's generated default that `nginx.conf` corrects:
   - `root`/`index` are declared only inside `location /`. Any further location
     block therefore falls back to the compiled-in root and 404s.
   - `try_files` lists `$uri/index.html` before `$uri/`, so `/ratgeber` returns
     the page body instead of redirecting to `/ratgeber/`. The browser then
     resolves the page's relative references (`./support.js`, `uploads/...`)
     against `/` and gets 404s — the page does not render at all. Verified.
2. **Direction** — currently *Allow www & non-www*. There is no DNS record for
   `www.ihreapotheken.htw-projekte.de`, so ACME validation for that hostname
   cannot succeed. Either set **Allow non-www only**, or add a `www` CNAME.
3. **Save → Redeploy.**

No environment variables, no health check path, no persistent storage needed.

Tip: before DNS is repointed, use *Generate Domain* to get a temporary
Coolify hostname and confirm the site renders there.

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
