# The cache boundary: two settings that multiply

This is the failure mode that cost the most time on this project, and it is worth writing down because the symptom looks like a healthy site.

## Symptom

An anonymous visitor loads the home page. It renders. HTML is valid, the byte count is plausible, the status is 200. Nothing looks wrong.

Then a logged-in user loads the same URL and receives **HTML that was rendered for someone else** — including that user's name in the navigation and their session-dependent links.

The page is not broken. It is *cached under the wrong identity*.

## Why it happens

Two independent settings, both of which have to be correct, and neither of which is visible from the origin:

| Setting | Where | Behaviour |
|---|---|---|
| Page Rule: *Cache Everything* + Edge TTL | Cloudflare → Rules → Page Rules | Puts HTML in the edge cache regardless of the origin's headers |
| Browser Cache TTL | Cloudflare → Caching → Configuration | Overrides the origin's `Cache-Control` when the origin sends a *lower* max-age. It only ever raises the value — it never lowers it. |

The second one is the trap. A `Page Rule` with a 2-hour Edge TTL explains a `max-age=7200` in the response. So does a zone-level Browser Cache TTL of 14400 with a 2-hour rule on top. If you only look at the response header, you cannot tell which one is responsible — and fixing the wrong one changes nothing.

Additional detail that matters: `private` and `must-revalidate` tokens survive the override. A response with `Cache-Control: private, no-cache, must-revalidate` is not rewritten. That means a naive test on an endpoint that already sets those tokens will pass while the rest of the site is still exposed.

## The test that actually distinguishes them

Pick endpoints that take different code paths through the rules:

```
/api/novels.php     matched by a Bypass Page Rule
                    → origin headers pass through untouched
                    → correct control: no-store, max-age=0

/                   matched by Cache Everything
                    → headers are rewritten
                    → this is where the override shows up
```

If the control endpoint shows the origin's value and the HTML endpoint shows a different, larger value, the rewrite is real. Compare `cf-cache-status` as well: `DYNAMIC` means the edge declined to cache — which can be *coincidental* (a `Set-Cookie` on the response is enough to prevent caching) rather than configured.

**A 302 to the login page is not evidence of safety.** Cloudflare does not cache 302s. The vulnerability only appears on authenticated requests that return 200 — exactly the requests a quick anonymous test never makes.

## The correct end state

Verify by header inspection, not by eye:

```
/            cf-cache-status: DYNAMIC   cache-control: no-cache
/chapter/*   cf-cache-status: DYNAMIC   cache-control: no-cache
/assets/*    cf-cache-status: HIT       cache-control: max-age=7200
/api/*       cache-control: no-store    (bypass rule)
```

Two properties, and both are required:

- **HTML must never be `HIT`.** Session-dependent documents cannot be shared across visitors.
- **Static assets should still be `HIT`.** A "fix" that turns everything into `DYNAMIC` has disabled the CDN instead of configuring it. The CSS row in the table above exists specifically to catch that overcorrection.

## Fix order

1. Page Rule for HTML paths: Cache Level → *Standard* (stop forcing Cache Everything).
2. Zone Browser Cache TTL → *Respect Existing Headers* (stop overriding the origin).
3. Purge the edge cache. Config changes do not evict what is already stored — removing a file or changing a rule leaves the old object serving until purged.
4. Re-measure. Expect HTML `DYNAMIC` and assets `HIT` in the same run.

Step 3 is the one people skip. A correct rule with a stale cache looks exactly like a broken rule.
