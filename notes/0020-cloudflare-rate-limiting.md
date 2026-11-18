# 0020 – Cloudflare rate limiting rule on `/api/*`

## Context

Not a product requirement. The point is to try out how Cloudflare rate limiting works, end to end, with Cloudflare already in front of every host (0018). The zone is on the Free plan.

## Decision

**One zone-level rate limiting rule**, `cloudflare_ruleset.rate_limit` in `infra/cloud/cloudflare.tf`: per IP, more than 100 requests to `/api/*` in 10 s → `block` for 10 s.

Copied from Cloudflare's Terraform example. It changes only what the Free plan forces:

- `matches "^/api/"` → `wildcard "/api/*"`. Regex (`matches`) needs Business. `wildcard` matches the whole path, so with a trailing `*` it's the same prefix match. One difference: `wildcard` ignores case and the regex doesn't. Express routing ignores case too, so `/API/hello` reaches the handler, and `wildcard` counts it. `strict wildcard` would be the exact equivalent of the regex.
- `period` 60 → 10 and `mitigation_timeout` 600 → 10. Free only allows 10 s for both.
- `requests_per_period` stays at the example's 100. With a 10 s period it's 6× looser than the example's rate. The number is arbitrary.

How it fits in:

- **Phase.** Rate limiting rules live in the `http_ratelimit` phase. It runs after `http_request_firewall_custom` (the `www` challenge, 0018) and before managed rules. Each zone has at most one entry point ruleset per phase, so all rate limiting rules go in this one resource.
- **Free plan limits.** One rule. The expression may only use path (no `http.host`), so the rule hits `/api/*` on every host. Only `api` serves `/api/`, so in practice that's only `api`. Counted by IP only.
- **`characteristics = ["cf.colo.id", "ip.src"]`.** `cf.colo.id` (data center) is mandatory in the API. Counters are kept per IP *and* per data center.
- **Token.** The rule needs `Zone WAF Write`, which `infra-ci` already has (0018).

Rejected:
- Account-level rate limiting rulesets: Enterprise only, and only useful for many zones.
- The old `cloudflare_rate_limit` resource: unsupported since 2025-06-15.
- Rate limiting in the cluster (Envoy Gateway etc.): the goal was to try the Cloudflare one.

## Consequences

- **The limit is per data center, not global.** "Counters are not shared across data centers." Verified: one burst of 150 parallel requests from one home IP in Vietnam was spread over SIN (111), HKG (26) and NRT (13). Only SIN went over 100, and only 4 requests got `429`. An earlier burst got no `429` at all, probably for the same reason (data centers weren't logged that time). One IP can get more than 100 requests per 10 s through when its traffic is split.
- **Counting isn't exact.** SIN let 107 through before blocking. Treat 100 as approximate.
- The block response is `429`, `text/plain`, with `retry-after` (9 s seen). A request ~12 s later got `200` again.
- Test (needs >100 requests within 10 s through one data center):
  `seq 150 | xargs -P 30 -I{} curl -s -o /dev/null -w "%{http_code} %header{cf-ray}\n" https://api.hienvuong.com/api/hello`
  The last part of `cf-ray` is the data center.
- `/livez` and `/readyz` are not under `/api/`, so they're not limited.
- `validate` doesn't check plan limits. Only the API checks them, at apply.

## References

- https://developers.cloudflare.com/waf/rate-limiting-rules/ (plan limits)
- https://developers.cloudflare.com/waf/rate-limiting-rules/parameters/
- https://developers.cloudflare.com/waf/rate-limiting-rules/request-rate/
- https://developers.cloudflare.com/terraform/additional-configurations/rate-limiting-rules/
- https://developers.cloudflare.com/ruleset-engine/rules-language/operators/
- https://developers.cloudflare.com/ruleset-engine/about/phases/
- https://developers.cloudflare.com/waf/reference/phases/
- https://developers.cloudflare.com/waf/account/rate-limiting-rulesets/
- https://developers.cloudflare.com/waf/reference/legacy/old-rate-limiting/upgrade/
