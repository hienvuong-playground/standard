# 0007 – HTTP caching for static assets

## Context

Moving images to Blob (0006) raised the question of how caching actually works. This note is mostly the explanation, so the `Cache-Control` values in the repo make sense later.

Follow-ups: 0008 (headers live at the origin; AKS/nginx and CDN notes), 0009 (never change content at a URL; arguments for hashed names).

## How caching works

- **There is cache without a CDN.** The browser is the first cache layer. `Cache-Control` is a response header that **whatever server returns the file** sends, and every cache on the way obeys it: the browser (private, per user) and a CDN or proxy (shared).
- **Order of lookup**: browser → CDN → origin.
  - Browser has a fresh copy (within `max-age`): no request at all (DevTools shows "disk cache").
  - Browser copy is stale: it revalidates (`If-None-Match`/`If-Modified-Since`) and gets `304`, no body.
  - CDN hit: served from the edge. Miss: the edge fetches from the origin and keeps a copy. Each edge POP has its own cache.
  - The CDN sends an `Age` header and the browser subtracts it from `max-age`, so a copy can't stay cached longer than `max-age` in total across both layers.
- **One header serves both layers.** To split them, use `s-maxage` (only shared caches read it), e.g. `max-age=60, s-maxage=86400`.
  - Front Door reads `s-maxage` → `max-age` → `Expires`. Without any of them it caches for a random 1–3 days. It never caches `no-store`/`no-cache`/`private`, even with a Rules Engine override.
  - A Rules Engine override changes only the edge TTL, not the header the browser gets.
  - Front Door revalidates with `Last-Modified` only (no `ETag`).
- **A CDN can be purged; browsers can't.** Once a browser holds a file under a long `max-age`, there's no way to make it refetch.
- **`no-cache` does not mean "don't cache".** The response is stored, but must be revalidated with the server before *every* reuse (`304` if unchanged). "Don't store" is `no-store`.
  - `max-age=N, must-revalidate` reuses the copy without asking for N seconds, then revalidates. `must-revalidate` also forbids serving a stale copy when the origin is unreachable.
  - Per MDN, `max-age=0, must-revalidate` is equivalent to `no-cache`.
  - **How long does a `no-cache` copy live? No limit, and none is needed.** Two separate things:
    - *Freshness* (how long it may be reused without asking) is 0 s. Adding `max-age` changes nothing, since every reuse is revalidated anyway.
    - *Storage* (how long it stays in the cache) isn't set by any header. The browser keeps it until it decides to evict: cache full (least recently used goes first), user clears data, incognito closes, low disk.
    - Each `304` refreshes the stored headers (RFC 9111 §4.3.4), so a frequently used `index.html` effectively stays forever.
    - Evicted is harmless: there's no `ETag` to send, so the next request is a plain `200` with the full HTML (a few KB), stored again.
    - Because every reuse asks the server, how long the copy lives never causes staleness. With a long `max-age` it does: the longer the copy lives, the higher the risk.
    - Shared caches: the spec lets a CDN store `no-cache` and revalidate each time, but Front Door doesn't cache it at all (see above). An edge "expiry" only exists with `s-maxage` + purge on deploy, and the browser still follows `no-cache`.
- **Revalidation cost is latency, not size.** A `304` is a few hundred bytes but still a full round-trip, plus TCP/TLS on a new connection. For `index.html` it blocks the whole page, because the browser can't know which JS/CSS to load yet.
  - Measured from Vietnam: the API in West Europe takes ~190–240 ms to connect and ~560–690 ms TTFB.
  - SWA is globally distributed ("static assets … are served from points geographically distributed around the world", SWA overview). The connection is close (~40 ms), but `index.html` TTFB is still ~340 ms. This suggests the edge fetches `no-cache` HTML from far away; that's an inference, the docs don't say.
- **A CDN without HTML caching only shortens the first leg.** Browser → edge is short (handshakes happen nearby), but edge → origin is unchanged. Caching HTML at the edge is what removes that long leg, so a CDN makes HTML edge caching *more* useful, not less.

## How others handle HTML (Jake Archibald's two patterns)

From "Caching best practices & max-age gotchas" (Jake Archibald, Chrome team), which this repo follows:

- **Pattern 1: immutable content + long max-age.** "You never change content at a particular URL, you change the URL." Versioned or hashed URLs, `max-age=31536000`. This is `/assets/*` and the Blob images.
- **Pattern 2: mutable content, always server-revalidated.** `no-cache`: "Any locally cached version isn't trusted without the server's say-so." Used for content whose URL can't be versioned, like HTML. Downside: "This pattern always involves a network fetch." This is `index.html`.
- **Don't mix a short `max-age` with mutable content.** It's "a baaaad compromise" that "can appear to work in testing, but break stuff in the real world". HTML, CSS and JS are interdependent, but "the caching headers can't express that". The browser can end up with old HTML/JS and new CSS. The old SWA default (`max-age=30` on HTML) was this pattern.
- Bottom line: "Favour immutable content for any URL that can easily change, otherwise play it safe with server revalidation."

web.dev ("HTTP cache") says the same about HTML: "This is one scenario in which you need to admit defeat. HTTP caching alone isn't powerful enough to avoid the network completely." It recommends `no-cache` with `ETag`, so the check is a cheap `304`.

**Decision: HTML is not cached at an edge, here or if Front Door is added later.** Edge caching of HTML saves the edge → origin leg (a few hundred ms for far users). The cost is a purge on every deploy, and a missed or failed purge serves stale HTML pointing at deleted assets, which breaks the app. Platforms like Vercel can do it safely because the cache is tied to each deployment and the platform handles invalidation ("Static files are automatically cached on Vercel's global network for the lifetime of the deployment"). Self-built, the purge is our pipeline's job. Some teams compromise with a short `s-maxage` (e.g. 60 s) and no purge, which still risks the version mismatch above.

If Front Door is ever added in front of SWA, things to remember:

- Routes:
  - `/assets/*` and Blob images: caching on, honor origin.
  - `/api/*`: caching off. The AFD docs recommend separate routes with caching disabled for dynamic content.
  - HTML: per the decision above.
- AFD **never caches `no-cache`/`no-store`/`private`**, even with a Rules Engine override (AFD caching doc). To cache HTML at the edge, the origin header would have to become something like `max-age=0, must-revalidate, s-maxage=86400`. That is still `no-cache`-equivalent for the browser, while AFD reads `s-maxage`.
- AFD caches per URL, so every deep link (`/product/3`, `/s?k=…`) is its own copy of `index.html`. A purge would have to be `/*`, not just `/index.html`. Use "Ignore Query String" on that route.
- Users can bypass AFD by hitting `*.azurestaticapps.net` directly. Blocking that (`forwardingGateway.requiredHeaders` with `X-Azure-FDID`) needs the SWA Standard plan.

## How SPA fallback works (and why `/assets/*` is excluded)

- **The frontend is an SPA.** SWA only holds `index.html` and the files under `assets/`. Pages like `/product/3` or `/s?k=headphones` are not files: React Router draws them in the browser from the URL.
- **`navigationFallback.rewrite: "/index.html"`** covers the case where a user opens such a URL directly (typed, F5, shared link). The server has no file at that path, so without the fallback it would return 404. With it, any path that matches no file gets `index.html` (`200`), the JS boots, and the router renders the right page. Every SPA needs this.
- **The fallback catches every missing path, including ones that must be real files.** A missing `/assets/index-OLD.js` would get `index.html` back as `200 text/html`:
  - The browser expects JS or an image and gets HTML. Instead of a clear 404, it fails with something like `Unexpected token '<'` or a broken image, while the network tab shows "success".
  - On SWA the `/assets/*` immutable header is attached to that response too (measured, see Consequences), so the HTML gets cached under that asset URL for a year.
- **`navigationFallback.exclude: ["/assets/*"]`** tells SWA that everything under `/assets/` is a real file: if it isn't there, return 404 and skip the fallback. This is safe because no app route lives under `/assets/`. All router paths (`/`, `/s`, `/product/:id`, `/cart`, …) are outside it and still use the fallback.
- **When does a missing asset get requested?** Typically a tab opened before a deploy. It still holds the old `index.html`, which points at the old hashed files. A deploy replaces the whole SWA content, so those files are gone. If the tab then loads another file (e.g. a lazy-loaded chunk), it asks for an old URL. With `exclude` it gets a clear 404, and a reload fixes it.

## Decision

**Every file referenced by code or data has a content-hashed name and `Cache-Control: public, max-age=31536000, immutable`. Entry points keep fixed names and use `no-cache`.**

- Why: with a fixed name plus long cache, replacing the file leaves returning users on the old version for up to a year, and nothing can fix it from the server. With a hashed name, new content means a new URL, so `immutable` is honest. Short cache plus revalidation would work for fixed names, but costs a `304` round-trip per file. This is MDN's "cache busting" pattern, with the same header value.
- `public` only matters if requests carry an `Authorization` header (it lets such responses be stored). They don't here.
- Fixed names are only for things whose URL others already hold: `index.html`, `robots.txt`, `favicon.ico`, the service worker, links sent out (emails, OG images). These use short cache plus revalidation.
- Each origin sets its own header:
  - **Blob** (content images): the `cache_control` property on each blob, in `infra/cloud/assets.tf`.
  - **Static Web Apps** (frontend build), in `frontend/public/staticwebapp.config.json`:
    - `globalHeaders` sets `Cache-Control: no-cache` on everything.
    - A route for `/assets/*` overrides that with `immutable`. The SWA docs say route headers override global ones.
- **Vite**: font and sprite moved from `public/` to `src/assets/` and are referenced relatively from `index.css`, so Vite hashes them. The Vite docs say to prefer imports; `public/` is only for files never referenced in code or that must keep their exact name.
- **`index.html` uses `no-cache`**, as MDN recommends for main resources ("Main resources" section): HTML URLs can't be cache-busted, so the browser must always check for a new one. Each page load costs one small `304` round-trip for `index.html`. In exchange, users get a new deploy on the next load, and the new `index.html` points at the new hashed files.
  - `globalHeaders` is used rather than a route on `/index.html` because deep links like `/s?k=...` and `/product/3` are served through `navigationFallback`, and the SWA doc says route rules don't apply there. (That note turned out to be inaccurate; see Consequences.)

## Consequences

- **SWA default, observed with `curl` and not documented**: `cache-control: public, must-revalidate, max-age=30` on every file. Before this change, hashed JS/CSS also got 30 s and were revalidated constantly. That worked, it just wasn't optimal.
  - For `index.html`, that default is `no-cache` plus 30 s of reuse without asking. It only helps if the page is reopened within 30 s, and it can show a stale `index.html` for up to 30 s after a deploy. Normal return visits revalidate either way, so `no-cache` costs about the same latency.
  - The `/assets/*` immutable route could not be added earlier, because `public/assets/` copied fixed-name images into `dist/assets/` and they would have become immutable too.
  - The rule only holds while `dist/assets/` contains hashed files only. **Never put fixed-name files back into `public/assets/`.**
- **Verified after deploy**: `globalHeaders` does apply to fallback responses. `/`, `/s?k=headphones` and `/product/3` all return `no-cache`.
- **Fixed: a missing `/assets/*` file used to return `index.html`, cached as immutable.** Measured before the fix: `/assets/does-not-exist.js` → `200`, `content-type: text/html`, `cache-control: public, max-age=31536000, immutable`. The fallback served `index.html`, and the `/assets/*` route header was applied anyway.
  - This **contradicts the SWA doc note** "Route rules aren't applied on requests that trigger `navigationFallback`" (section "Fallback routes"). Trust the measurement, not the note.
  - Effect: a browser that ever requested a non-existent asset URL (e.g. the old `/assets/nav-sprite.png`, or an old chunk after a deploy) would have stored HTML under that URL for a year.
  - Fix: `"exclude": ["/assets/*"]` in `navigationFallback`, as in the doc's example (`"exclude": ["/images/*.{png,jpg,gif}", "/css/*"]`, where a missing file under an excluded path returns 404).
  - Verified after deploy: missing `/assets/*` files return `404` with **no** `Cache-Control`. Existing hashed files still return `immutable`, and `/` plus deep links still return `no-cache`.
- To verify after deploy:
  - `curl -I` on `/` **and on a deep link** (`/s?k=headphones`) shows `no-cache`. If the deep link doesn't, `globalHeaders` doesn't cover the fallback.
  - `curl -I` on an `/assets/*.js` file shows `immutable`.
  - `curl -I` on a blob URL shows `immutable` and `image/jpeg`.

## References

- https://vite.dev/guide/assets
- https://learn.microsoft.com/en-us/azure/static-web-apps/configuration (sections "Routes", "Global headers", "Fallback routes")
- https://learn.microsoft.com/en-us/azure/static-web-apps/overview (global distribution)
- https://jakearchibald.com/2016/caching-best-practices/
- https://web.dev/articles/http-cache
- https://vercel.com/docs/caching/cdn-cache
- https://learn.microsoft.com/en-us/azure/frontdoor/front-door-caching
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Caching (sections "Cache busting", "Main resources")
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control
- https://www.rfc-editor.org/rfc/rfc9111.html (§4.3.4 freshening stored responses on 304)
