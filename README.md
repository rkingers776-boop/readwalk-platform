# ReadWalk

A multi-author web-novel publishing platform. Plain PHP 8, MySQL 5.7, nginx — no framework, no build step, no JS framework on the server side.

**Live site:** [readwalk.com](https://readwalk.com)

---

## What this is

ReadWalk is a self-hosted serial-fiction platform: authors register, create a series, publish chapters on a schedule, and readers follow along with bookmarks, reading history and comments. It runs on a single VPS behind Cloudflare, with no external services for auth, storage or payments.

Everything is server-rendered PHP. The only client-side JavaScript is progressive enhancement (theme toggle, in-page search filtering, the subscribe form). The site works with JavaScript disabled.

I am publishing the platform design here because the codebase is more interesting than the catalogue it currently serves, and because a platform with a public spec is easier to hand over or build on than a pile of private scripts.

---

## Stack

| Layer | Choice | Why |
|---|---|---|
| Runtime | PHP 8.3 (php-fpm) | Ubiquitous on cheap shared hosting; no deploy artifact needed |
| Database | MySQL 5.7 | Present on the host; InnoDB, utf8mb4 |
| Web server | nginx 1.21 | Serves static assets directly, proxies `.php` to php-fpm |
| Edge | Cloudflare | TLS, and a cache boundary that has to be configured carefully (see below) |
| CSS | Tailwind, precompiled | One stylesheet, no runtime JIT |
| Auth | Session cookies + `password_hash` | No third-party identity provider |

There is no Composer dependency tree and no `node_modules` at runtime. The deploy artifact is a zip of `.php` files.

---

## How a chapter gets published

The pipeline is deliberately boring, because boring is what keeps a five-year-old install running:

1. **Author submits** a chapter through `/write` — title, body as plain text, optional publish date.
2. **Word count is derived, never stored from user input.** A single `count_words()` helper is the only authority, so listings, series totals and sitemap priorities can never disagree with the reader's view.
3. **Scheduled chapters** sit with `status = 'scheduled'` and a future `published_at`. A cron endpoint promotes everything whose time has passed. If the endpoint is called twice, nothing is published twice — the update is `WHERE status = 'scheduled' AND published_at <= NOW()`.
4. **Cron is closed by default.** It requires a shared key and returns 503 when no key is configured, rather than treating "unconfigured" as "unprotected". A scheduler that is anonymously triggerable is a write endpoint with no auth, and it is easy to ship that by accident.
5. **Caching is invalidated by version, not by time.** The home page renders into a small cache file carrying an integer version. Publishing a chapter increments the version; the next request rebuilds. Nothing waits for a TTL to expire.

### The cache boundary, which is the part people get wrong

Two independent settings multiply here, and both have to be right:

- A Cloudflare **Page Rule** with *Cache Everything* plus an Edge TTL will put HTML responses in the edge cache regardless of what the origin says.
- The zone-level **Browser Cache TTL** setting overrides the origin's `Cache-Control` when the origin sends a lower value, and it only ever raises the value — it does not lower it.

The failure mode is not a broken page. It is a **cached page in the wrong session state**: an anonymous visitor receives HTML that was rendered for a logged-in user, complete with their navigation. Measured output looks completely healthy — 200, correct byte count, valid HTML.

The correct end state, verified by header inspection rather than by eye:

```
/            cf-cache-status: DYNAMIC   cache-control: no-cache
/assets/*    cf-cache-status: HIT       cache-control: max-age=7200
/api/*       cache-control: no-store    (bypass rule)
```

HTML must never be `HIT`. Static assets should be. A rule that fixes the first by disabling the second is not a fix.

---

## Data model

Eleven tables. The interesting decisions are in three places.

```sql
-- Chapters belong to a series; ordering is explicit, never by id.
CREATE TABLE chapters (
  id            INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  novel_id      INT UNSIGNED NOT NULL,
  chapter_number INT UNSIGNED NOT NULL,
  title         VARCHAR(255) NOT NULL,   -- stores the full line, e.g. "Chapter 7: The Debt"
  body          MEDIUMTEXT NOT NULL,
  word_count    INT UNSIGNED NOT NULL,
  status        ENUM('draft','scheduled','published') NOT NULL DEFAULT 'draft',
  published_at  DATETIME NULL,
  deleted_at    DATETIME NULL,
  UNIQUE KEY uniq_novel_chapter (novel_id, chapter_number)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**1. Soft delete is applied asymmetrically, on purpose.**

A `deleted_at` column exists on series and chapters. Reader-facing and SEO-facing reads filter on it. Write paths deliberately do **not**, because uniqueness checks and "next chapter number" must see deleted rows — otherwise deleting chapter 7 makes the next publish collide, or worse, reuse the number.

Getting this backwards in either direction produces a bug that only appears after the first deletion, which is exactly the kind of bug that survives a year in production.

**2. Aggregate queries must tolerate empty sets.**

`SUM()` over zero rows returns `NULL`, not `0`. A dashboard that renders `SUM(word_count)` through a typed formatter will throw a `TypeError` the first time it meets an author with no published work. Every aggregate in the codebase is therefore either `COALESCE`d or cast at the display boundary.

This is not hypothetical: it is how the author dashboard returned HTTP 500 to a newly registered account with zero series.

**3. `novels.updated_at` is not touched by chapter publication.**

Touching it on every publish would make a decade-old series jump to the top of the "recently updated" list and push genuinely active work off the page. Sort keys and activity signals are computed from chapter timestamps instead.

---

## The catalogue

57 series, 896 chapters (763 published), roughly 1.66M words as of November 2026. Series that have stopped are labelled `hiatus` rather than `ongoing` — the label is derived from actual activity, not aspiration.

A representative sample, each linked at its opening chapter:

| Series | Genre | Chapters | Status |
|---|---|---|---|
| [Gravity of the Frozen Will](https://readwalk.com/novel/gravity-of-the-frozen-will/chapter/1) | Progression / Dark Fantasy | 28 | Complete |
| [Forty Second Loop: Execution Protocol](https://readwalk.com/novel/forty-second-loop-execution-protocol/chapter/1) | Time Loop / Military SF | 21 | Complete |
| [The Heretical Canvas](https://readwalk.com/novel/the-heretical-canvas/chapter/1) | Dark Fantasy / Art Magic | 19 | Complete |
| [The Flawed Canon](https://readwalk.com/novel/the-flawed-canon/chapter/1) | Progression / Legal Thriller | 15 | Hiatus |
| [Osseous Calibration: Neural Necromancy](https://readwalk.com/novel/osseous-calibration-neural-necromancy/chapter/1) | Crafting LitRPG / Grimdark | 15 | Hiatus |
| [System Deadlock: The Crashed Core](https://readwalk.com/novel/system-deadlock-the-crashed-core/chapter/1) | LitRPG / System Apocalypse | 16 | Hiatus |
| [The Last Blackthorn Exile](https://readwalk.com/novel/the-last-blackthorn-exile/chapter/1) | Dark Fantasy / Anti-System | 15 | Hiatus |
| [Whispers of the Tobacco Rows](https://readwalk.com/novel/whispers-of-the-tobacco-rows/chapter/1) | Cozy Historical / Quiet Resistance | 16 | Hiatus |
| [Tending the Unscathed Garden](https://readwalk.com/novel/tending-the-unscathed-garden/chapter/1) | Cozy Post-War / Healing | 16 | Hiatus |
| [No Blades, Only Bloom](https://readwalk.com/novel/no-blades-only-bloom/chapter/1) | Cozy Fantasy / Domestic | 15 | Hiatus |
| [Moonlight Tears](https://readwalk.com/novel/moonlight-tears/chapter/1) | Fantasy / Romance | 16 | Ongoing |

Catalog-level pages:

- [All series](https://readwalk.com/novels/) — filterable by genre, tag and status
- [Maintenance log](https://readwalk.com/changelog) — a dated record of what actually changed on the site, including the periods where nothing changed

---

## Honest status (November 2026)

The platform works. The catalogue does not currently have an audience. Organic traffic is effectively zero: search crawling has not discovered a single new URL on this domain in the last two quarters, while the few requests it does make go to URLs inherited from a previous owner.

Two things follow from that, and both are recorded here rather than hidden:

- **The design is the asset.** A working multi-author platform with auth on every write endpoint, scheduled publishing, soft delete, cover upload with content-type sniffing and re-encoding, and a documented cache boundary — that is the thing worth reading.
- **Adding content will not fix distribution.** Publishing more chapters into a domain that receives no crawl budget changes the size of the archive, not the number of readers. The site is in maintenance mode, not growth mode.

If you are evaluating this repository as a potential acquirer or collaborator, that paragraph is the due diligence.

---

## Operational notes

- **Deploys are zip artifacts, not directory syncs.** Syncing a working directory publishes audit reports, local notes and one-shot migration scripts along with the code. The release script builds an explicit whitelist and asserts that the credentials file is absent from the archive before it will produce output.
- **`.htaccess` is inert.** The origin's SAPI is nginx; nothing reads Apache config. Any access rule that only exists in `.htaccess` is documentation, not enforcement.
- **SEO surfaces are generated, not hand-maintained.** The sitemap is assembled from live rows plus a static page table; there is no checked-in XML file to drift out of date.
- **No ads, no analytics scripts beyond a page-view counter, no paywall.** Reader accounts, bookmarks and history are stored; nothing is sold.

---

## What is not in this repository

Deliberately excluded: database credentials, the production `nginx.conf`, the audit and incident write-ups, and the deployment packages themselves. If you want the schema or a specific subsystem explained, open an issue — the point of publishing the design is to make it discussable.
