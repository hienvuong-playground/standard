# 0008 – Cache headers are set at the origin; CDNs only honor them

## Context

0007 set the cache headers on SWA (`staticwebapp.config.json`) and on Blob (`cache_control` per blob). Follow-up questions came up: where would the headers live if the frontend moved into AKS, and does adding a CDN (Front Door, Cloudflare) mean configuring caching again? Also "HTML is not cached at an edge" was misread as "cached at the origin".

## Decision

**`Cache-Control` is decided by the server that returns the file (the origin). Anything in front (gateway, CDN) passes it through or honors it and sets no TTLs of its own.** Swapping or adding a CDN then doesn't change caching behavior.

### Terms

- **Origin** = where the real file lives (SWA build output, Blob, an nginx pod). It is not a cache layer; every miss ends there.
- **Caches** = copies between the user and the origin: the browser (per user) and the CDN edge (shared).
- For `index.html` today:
  - Origin holds the file.
  - The edge keeps no copy.
  - The browser stores it but, because of `no-cache`, revalidates before every use (`304` if unchanged).

### If the frontend is hosted in AKS

The origin becomes the web server in the frontend container, typically nginx serving `dist/`. The same behavior as `staticwebapp.config.json`:

```nginx
location /assets/ {
  add_header Cache-Control "public, max-age=31536000, immutable";
  try_files $uri =404;              # = navigationFallback.exclude
}
location / {
  add_header Cache-Control "no-cache";   # = globalHeaders
  try_files $uri /index.html;            # = navigationFallback.rewrite
}
```

- nginx sends `ETag`/`Last-Modified` for static files, so `no-cache` costs a small `304`.
- `add_header` only applies to 2xx/3xx by default, so a missing asset's 404 has no `Cache-Control`. This matches what was verified on SWA.
- A `location` that has its own `add_header` does **not** inherit `add_header` from `server`. This is a common trap when security headers are added later at server level.
- **Not at the Istio Gateway** (`HTTPRoute` `ResponseHeaderModifier`). It can set headers per path, but it can't tell whether a file exists. That is exactly how SWA ended up putting `immutable` on fallback HTML (0007). It would also split fallback and cache config across two places.

### If a CDN is added later

Both Front Door and Cloudflare honor origin `Cache-Control` by default. Neither caches `no-cache`/`no-store`/`private`. With the current headers, `/assets/*` and Blob images get edge-cached, and HTML does not, which is the decision in 0007.

What still has to be set on the CDN side:

| | Front Door | Cloudflare |
|---|---|---|
| Turn caching on | Per route, opt-in ("Caching: Select this option to enable caching") | Default caches by **file extension** only (JS, CSS, PNG, JPG, WOFF2, …); "does not cache HTML or JSON by default" |
| API | Separate `/api/*` route with caching off | JSON not cached by default; API also sends `no-store` |
| Query strings | Choose Ignore/Use on cached routes | Own default |
| **Don't** | Rules Engine "Override always", which ignores the origin TTL | Edge Cache TTL rules, which "override the headers" |

## Consequences

- One place decides caching per origin. Moving SWA → AKS means translating `staticwebapp.config.json` to `nginx.conf` (table above). Adding or switching a CDN means turning caching on, not redefining TTLs.
- If someone adds a TTL override on the CDN, the CDN and the origin disagree and the notes no longer describe reality. Check `Cache-Control` and the CDN cache status header (`X-Cache` on AFD, `cf-cache-status` on Cloudflare) when debugging.

## References

- https://learn.microsoft.com/en-us/azure/frontdoor/front-door-caching
- https://learn.microsoft.com/en-us/azure/frontdoor/how-to-configure-endpoints (route "Caching" option)
- https://developers.cloudflare.com/cache/concepts/default-cache-behavior/
- https://nginx.org/en/docs/http/ngx_http_headers_module.html#add_header
