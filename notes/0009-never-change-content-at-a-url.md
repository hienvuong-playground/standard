# 0009 – Never change content at a URL (why images get hashed names)

## Context

Content images were given hashed blob names (0006) so they can be cached as immutable (0007). This note is the general rule behind that. It's written so it can be used as an argument if a team stores images under fixed names like `headphones.jpg` and overwrites them.

## Decision

**Never change the content at a URL. To change content, change the URL.** A content hash in the name is one way to guarantee it automatically. A UUID per upload, or a version suffix, also works.

Sources:

- MDN (HTTP caching, "Cache busting"): "for resources that do change, it is a common best practice to change the URL each time the content changes, so that the URL unit can be cached for a longer period." It "can be applied to all subresources, such as images."
- Jake Archibald (Caching best practices): "you never change content at a particular URL, you change the URL". Also: "Favour immutable content for any URL that can easily change, otherwise play it safe with server revalidation."
- web.dev (HTTP cache): once a file is cached long, an urgent fix is impossible "at least not without changing the URL of the resource". Without versioning, "different users might end up using different versions of the file."

### The argument

A fixed name that **gets overwritten** forces a choice between three options:

1. **Long cache**: users keep the old image, and nothing server-side can fix it. A CDN can be purged; browsers can't.
2. **`no-cache`**: a revalidation request per image on every page view. Blob's `ETag`/`Last-Modified` keep it a small `304`, but it's still a round-trip each.
3. **Short `max-age`**: stale for up to that long, then requests again.

A new URL per change avoids the trade-off: maximum caching and instant updates.

### Fair counterpoints

- **"The name is fixed but we never overwrite; a new image gets a new name."** That already follows the rule, so `immutable` is fine. A hash just enforces it instead of relying on discipline.
- **Readable names / SEO**: keep both, e.g. `headphones-7a2e1d51.jpg` or `7a2e1d51/headphones.jpg`.
- **The URL is already held by others** (emails, partners, OG/share images): a fixed name is required. Use `no-cache` or a short cache. This is the legitimate exception, together with `index.html`, `robots.txt` and `favicon.ico`.
- **Migration cost**: real. The rule can be applied to new uploads only.
- **Hash vs UUID**: upload systems usually use a UUID per upload. A content hash also dedups identical files, which is why it fits here: products reuse the category/promo images.

### Fixed names that do get overwritten: which header

- Images aren't interdependent like HTML/CSS/JS, so a stale image is only wrong for a while; it doesn't break the app. Both `no-cache` and a short `max-age` (1 h, 1 day) are defensible; it's a freshness vs request-count choice. Jake's warning against short `max-age` is about interdependent resources.
- With a CDN, overwriting also needs a purge, unless the header is `no-cache` (the CDN doesn't cache it).
- Middle way: keep the name, version the URL with a query, e.g. `headphones.jpg?v=<etag or updated_at>`. MDN lists this under cache busting. The CDN cache key must include the query string.

## Consequences

### Hashing doesn't make stale images impossible

Staleness moves from the file to the data. With hashed names, the bytes at a URL are never stale. But if whatever **holds the link** is stale, users still see the old image even though that file is correct for its URL. Examples of stale link holders: an API response, rendered HTML, an app-side cache, a lagging replica.

- Example: the observation that Facebook sometimes shows an old profile picture after it's changed. Its image URLs are long and unique per image, so it's likely not fixed-name overwriting but stale copies of data: feed caches, pre-rendered posts/comments, cross-datacenter eventual consistency, app caches. This is an inference; Facebook's internals are unknown.
- Here the data layer is always fresh: `/api` responses are `no-store` (`backend/src/app.js`), and the frontend fetches image URLs from the API on each load.

### What "cached for a year" means in the browser

- `max-age=31536000` is the **maximum** the browser may reuse without asking, not a guarantee. Browsers evict under size pressure, and rarely viewed images go sooner.
- A changed file means a new URL. The old entry isn't invalidated; it's just never used again and eventually evicted.

| Situation | Cache |
|---|---|
| Close and reopen the browser | Kept (disk cache). Only the memory cache is lost |
| Incognito | Separate, starts empty, deleted when all incognito windows close |
| Clear browsing data | Gone |
| Normal reload | Still used. `immutable` prevents revalidation |
| Hard reload (Ctrl/Cmd+Shift+R), DevTools "Disable cache" | Bypassed |
| Other profile / device | Separate |

- Browsers **partition the HTTP cache per top-level site**. The same image URL embedded by two sites is cached twice. That's why "use a shared public CDN to reuse users' existing cache" no longer holds. It doesn't matter here (one site).

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Caching
- https://jakearchibald.com/2016/caching-best-practices/
- https://web.dev/articles/http-cache
