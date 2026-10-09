# Staged migration to Vercel with interim rewrites

This example shows how to move a site to Vercel without making a DNS change on go-live day. The cutover happens in config, not DNS, so rollback is a redeploy instead of a DNS edit.

Three hostnames are involved:

| Hostname | Role |
| --- | --- |
| `new.example.com` | The new application, deployed on Vercel |
| `legacy.example.com` | The existing production site, served from your old infrastructure |
| `retired.example.com` | An old domain being retired, whose traffic should land on the new site |

## How the rollout works

**Phase 1, before go-live.** Deploy with `vercel.json` as it ships in this repo. Every request to your project, including the production `*.vercel.app` URL, is rewritten to the legacy origin. The Vercel project now mirrors the live site. You can point the production domain at Vercel days ahead of go-live, at a quiet time, and nothing changes for visitors because Vercel serves exactly what the old infrastructure serves.

**Go-live day.** Replace `vercel.json` with `vercel.go-live.json` (rename it to `vercel.json`) and deploy. Requests for the new locale subtree, `en-in/` here, now hit the new application. Everything else keeps proxying to the legacy origin, so the rest of the site keeps working while it migrates. If something looks wrong, deploy the previous config again; the old behavior is back in seconds, with no DNS involved.

**Later.** As more of the site migrates, delete the rewrite entirely. The legacy origin can be retired when the last locale moves.

## The two configs

`vercel.json` (interim, deploy first):

```json
{
  "$schema": "https://openapi.vercel.sh/vercel.json",
  "rewrites": [
    {
      "source": "/:path*",
      "destination": "https://legacy.example.com/:path*"
    }
  ]
}
```

`vercel.go-live.json` (rename to `vercel.json` on go-live day):

```json
{
  "$schema": "https://openapi.vercel.sh/vercel.json",
  "rewrites": [
    {
      "source": "/:path((?!en-in/).*)",
      "destination": "https://legacy.example.com/:path*"
    }
  ],
  "redirects": [
    {
      "source": "/:path*",
      "has": [{ "type": "host", "value": "retired\\.example\\.com" }],
      "destination": "https://new.example.com/en-in/:path*",
      "permanent": true
    }
  ],
  "headers": [
    {
      "source": "/:path*",
      "has": [{ "type": "host", "value": "retired\\.example\\.com" }],
      "headers": [{ "key": "X-Robots-Tag", "value": "noindex" }]
    }
  ]
}
```

What the go-live config does:

- **Rewrite** — a negative lookahead, `(?!en-in/)`, makes the catch-all match everything *except* paths starting with `en-in/`. Those requests proxy to the legacy origin. The new app handles `en-in/` paths itself.
- **Redirect** — requests arriving with the retired hostname are permanently redirected (308) to the new domain with the path preserved. The `has` / `host` condition scopes the redirect to that hostname only, so other domains and preview URLs are untouched.
- **Header** — requests on the retired hostname get `X-Robots-Tag: noindex`, so search engines drop the old URLs instead of ranking them alongside the new ones.

If the old domain is not being retired at cutover, ship the rewrites first and add the `redirects` and `headers` blocks as their own change when it retires. Redirects run before rewrites, so the retired-domain redirect applies even while the catch-all rewrite is in place.

One edge case to check: the lookahead excludes `en-in/` with a trailing slash. A bare `/en-in` (no trailing slash) still matches the rewrite and proxies to the legacy origin. Test it, and adjust the pattern to `(?!en-in($|/))` if that matters for your routing.

This example is a static page for simplicity. The same `vercel.json` approach works with any framework. Rewrites to external origins are handled at Vercel's edge, before your application runs, so they apply regardless of framework and add no compute cost. If your application is Next.js, keep using `next.config` for application-internal routes, and put external-origin rewrites in `vercel.json`.

## Testing before any DNS change

Test everything on the production `*.vercel.app` domain first:

1. Deploy phase 1 and open `https://<your-project>.vercel.app/`. You should see the legacy site, not the new app. If you see the new app's placeholder, the rewrite is not matching.
2. Spot-check a handful of deep paths from the legacy site, including ones with query strings.
3. Spot-check a deep path and one with a query string using curl:

```sh
curl -sI 'https://<your-project>.vercel.app/some-path?ref=test'
```

If the retired domain is already added to the project, check its redirect with `curl -sI 'https://retired.example.com/some-path'` once its DNS resolves to Vercel; expect a 308 and a `Location` header pointing at the new domain with the same path.

4. Move the production domain to Vercel (days before go-live, per phase 1) and confirm visitors see no change. DNS records to point at Vercel are shown in the project's **Settings → Domains** when you add the domain.
5. On go-live day, deploy the go-live config. Verify `en-in/` serves the new app and other paths still serve legacy content.

`public/index.html` in this repo is a placeholder page that says as much. It is what you see whenever the rewrite is not active.

## Retiring the old domain

The permanent redirect and noindex header for the retired hostname ship in `vercel.go-live.json`. When an old domain should stop serving entirely and its traffic should land on the new site, add it to the Vercel project and use a hostname-scoped redirect plus a noindex header, so visitors land in the right place and search engines stop ranking the old URLs. See the go-live config above for the pattern.

The `has` / `host` condition scopes both rules to that hostname only, so other domains and preview URLs are untouched. Status code 308 tells search engines the move is permanent. If the old domain's retirement is not decided yet, move the `redirects` and `headers` blocks out of `vercel.json` and add them in a separate deploy once it is.

## Third-party CDN in front of Vercel

If a CDN sits in front of Vercel (Fastly is the common case, since several CMS platforms ship with it), Vercel's firewall loses sight of the real client IP unless the CDN forwards it. Fastly does this with the built-in `Fastly-Client-IP` header, which Vercel reads automatically; no extra configuration is needed on the Vercel side.

Two prerequisites from the [reverse proxy docs](https://vercel.com/docs/security/reverse-proxy) matter for the CDN team:

- Do not cache `https://<DOMAIN>/.well-known/vercel/*` paths.
- Do not redirect HTTP to HTTPS for `http://<DOMAIN>/.well-known/acme-challenge/*` on port 80.

Skipping these breaks certificate validation and Vercel's own health checks. Verify in staging that the CDN actually forwards the client IP header before go-live, and watch the Firewall section in your Vercel dashboard for denied or flagged requests afterwards.

## SSL certificates

Vercel issues and renews certificates automatically for every domain added to a project, including domains where DNS is managed elsewhere. There is nothing to upload for the standard case.

Enterprise teams can upload their own certificates, which is what you would do to pre-stage a certificate for a domain before its DNS moves, or to keep using a certificate your security team manages. Renewal of uploaded certificates is then your responsibility. See [uploading custom SSL certificates](https://vercel.com/docs/domains/custom-ssl-certificate).

## Limits

- `vercel.json` supports up to 2,048 rewrites and redirects. Regex patterns cover far more URLs than a one-rule-per-URL list.
- For very large redirect lists, [bulk redirects](https://vercel.com/docs/routing/redirects/bulk-redirects) load thousands of rules from a file.
- Dashboard route rules are an alternative but live outside version control; configs in git give you history and one-command rollback.

## Optional hardening

When proxying to your own legacy origin, you can make Vercel attach a secret header to proxied requests and have the origin reject anything without it. That stops people from bypassing Vercel and hitting the old origin directly. See [restricting your origin to Vercel traffic](https://vercel.com/docs/routing/rewrites#restricting-your-origin-to-vercel-traffic).

## Docs

- [Rewrites](https://vercel.com/docs/routing/rewrites) and [redirects](https://vercel.com/docs/routing/redirects)
- [Adding a custom domain](https://vercel.com/docs/domains/working-with-domains/add-a-domain)
- [Reverse proxy servers and Vercel](https://vercel.com/docs/security/reverse-proxy)
- [Incremental migration](https://vercel.com/docs/incremental-migration)
- [Zero-downtime migration guide](https://vercel.com/kb/guide/zero-downtime-migration)

## Deploy this example

```sh
vercel
```

The project deploys as a static site. Replace `public/index.html` with your application and the configs above with your real hostnames.
