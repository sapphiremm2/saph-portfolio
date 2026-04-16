# saph — a self-hosted Linktree for content creators

**Live:** [saph.one](https://saph.one) · **Source:** private

saph is a link-in-bio platform I built for myself. It started as a weekend "Linktree clone" and grew into a full multi-user creator hub — per-page theming, live integrations with YouTube / TikTok / Roblox / Spotify / Discord, a URL shortener with QR codes, edge-rendered rich embeds, and first-class analytics.

This is a case study of how it's built. The source code is kept private but the architecture, tech choices, and interesting engineering problems are all here.

---

## Screenshots

| Public page | Admin panel | Rich Discord embed |
|---|---|---|
| ![public](./screenshots/public.png) | ![admin](./screenshots/admin.png) | ![embed](./screenshots/embed.png) |

---

## What it does

- Hosts **multiple pages** under one domain — `saph.one/` for me, `saph.one/fwnsie` for my sister, etc.
- Each page has its own **theme** (colors, animated backgrounds, card style, particle effects), its own set of links, its own integrations, and its own analytics.
- **Live integrations** pull real-time data:
  - YouTube — subscribers, views, latest video
  - TikTok — follower count, likes
  - Roblox — profile stats, group memberships
  - Spotify — "now playing" widget via OAuth + refresh token rotation
  - Discord — presence (online/idle/dnd), custom status, Spotify track
- A built-in **URL shortener** with QR codes (`saph.one/aB3xY`)
- An **analytics** dashboard — views, clicks, geo breakdown, top referrers
- **Rich link embeds** — when you paste a saph link in Discord / Twitter / Slack, it renders a proper preview card with the page's theme color, avatar, and bio
- **Multi-user** — I'm the admin, my sister is an editor; RLS policies enforce that editors only see their own pages

---

## Tech stack

| Layer           | Tech                                                       |
|-----------------|------------------------------------------------------------|
| Frontend        | React 19 · Vite · TypeScript · Tailwind CSS v4             |
| Animation       | framer-motion v12                                          |
| Hosting         | Cloudflare Workers + Assets                                |
| API             | Cloudflare Worker (separate deploy)                        |
| Database + Auth | Supabase (Postgres + RLS + Auth)                           |
| Secrets & state | Cloudflare KV (OAuth refresh tokens)                       |
| Scheduling      | Cloudflare Cron Triggers                                   |

Two workers:
- **`saph`** — frontend at `saph.one`. Serves the SPA, and intercepts link-preview crawlers (Discord, Twitter, Slack, ~20 user-agents) to inject dynamic Open Graph meta tags.
- **`saph-worker`** — API for OAuth flows, third-party API calls, analytics ingestion, and scheduled syncs.

---

## Architecture

```
 ┌──────────────┐     ┌─────────────────┐
 │  Your phone  │────▶│  Cloudflare CDN │
 └──────────────┘     │   (saph.one)    │
                     └────────┬─────────┘
                              │
              ┌───────────────┴───────────────┐
              │                               │
              ▼  (bot UA detected)            ▼  (normal browser)
      ┌────────────────┐               ┌────────────────┐
      │ OG meta inject │               │  SPA assets    │
      │ worker (edge)  │               │  (dist/)       │
      └───────┬────────┘               └────────┬───────┘
              │                                 │
              └──────────────┬──────────────────┘
                             │
                             ▼
                   ┌──────────────────┐
                   │   Supabase       │
                   │   (Postgres+RLS) │
                   └────────┬─────────┘
                            │
                            │ private service key
                            ▼
                   ┌──────────────────┐
                   │  API worker      │
                   │  (saph-worker)   │◀────── YouTube / TikTok / Roblox / Spotify
                   └────────┬─────────┘
                            │
                            ▼
                   ┌──────────────────┐
                   │  Cloudflare KV   │
                   │  (refresh tokens)│
                   └──────────────────┘
```

A few architectural choices worth calling out:

- **The frontend worker does dual-duty.** Real browsers get the SPA untouched. But when Discord or Twitterbot requests the page, the same worker intercepts, fetches the page row from Supabase (with a 60s edge cache), and returns server-rendered HTML with proper OG tags. Zero added latency for human visitors, rich previews for bots.
- **Two bundles via code splitting.** The public page is the hot path (every visitor hits it) so `React.lazy` pulls the admin panel, terms, and privacy pages into separate chunks. Public bundle ships in ~120 KB gzipped; admin is a separate 48 KB that only logged-in users pay for.
- **Edge caching for third-party APIs.** Roblox and YouTube upstream calls are wrapped in `fetch(url, { cf: { cacheTtl, cacheEverything: true } })`. A page visited a thousand times an hour hits Roblox's servers maybe twice.
- **FNV-1a hashing for IP dedup.** View tracking needs uniqueness but not identifiability — raw IPs are never stored, just their hash.

---

## Interesting problems I hit

### 1. The light blue flash
On first paint, there was a jarring light-blue flash before React rendered and applied the page's theme. The fix: set the HTML `body` background to `#0a0a0c` by default, then use `html:has(body.admin)` to switch to the warm off-white for the admin panel only. The public page overrides it once the theme loads. No flash, no CLS.

### 2. OAuth refresh tokens in a stateless worker
Spotify's "now playing" endpoint needs a fresh access token every hour. But the worker has no persistent memory. Solution: store the **refresh token** in Cloudflare KV (per-user), and on each API call, spend it for a new access token, use that access token, and discard it. Spotify's auth flow returns a new refresh token on each exchange, so rotation happens for free.

### 3. Slug collisions between pages and short links
`saph.one/fwnsie` is a page. `saph.one/aB3xY` is a short link. But they share the same namespace. The router checks `short_links` first (it's a redirect if hit), falls through to `pages` otherwise. And page creation + short link creation both validate against the other table at insert time, so you can't accidentally shadow. Plus a `RESERVED = ['admin', 'terms', 'privacy', 'api']` list for routes that would shadow app paths.

### 4. Rich embeds without SSR
I didn't want to migrate to Next.js just for OG tags. Instead, the frontend worker inspects the `User-Agent` header against a regex of ~20 known crawlers (`discordbot`, `twitterbot`, `slackbot`, etc.). For bots, it skips the SPA entirely and returns a minimal HTML document with server-rendered OG + Twitter Card meta, pulling the page data directly from Supabase REST. Cloudflare caches the bot response for 60s so successive crawler hits are free.

### 5. Sparkles that repel from the cursor
My sister wanted ambient sparkles clustered at the edges of her page (clear in the middle), reacting to her cursor. Each sparkle has a "home" position (placed via rejection-sampling an elliptical vignette), a current position, and a simple spring: `current += (target - current) * 0.18`. The target is the home, unless the cursor is within 130px — in which case it's offset outward with a force that scales inversely with distance. The whole thing runs on one `<canvas>` in a single `requestAnimationFrame` loop.

### 6. Multi-user without over-engineering
Two users, one as admin, one as editor. I didn't want a whole RBAC system. The whole access layer is:
- A `profiles(user_id, role)` table
- A single `is_admin()` SQL function
- RLS policies: `USING (is_admin() OR owner_id = auth.uid())`
- A trigger that auto-creates a `profiles` row with `role='sub'` on signup
- Role escalation is impossible from the client — no policy allows updating `profiles.role`
- The frontend reads its own role, hides admin-only tabs

~80 lines of SQL, hard to misuse.

---

## Lessons learned

- **Edge-first beats server-first for small apps.** Cloudflare Workers + Supabase + KV is genuinely enough infrastructure for most creator-tool ideas. No containers, no VPCs, no servers.
- **RLS is underrated.** Putting authorization in the database means the frontend can't get it wrong — worst case, the API returns empty results.
- **The "just check the User-Agent" trick for link previews is amazing.** No SSR, no static generation, no build-time complexity. Bots get HTML, humans get the SPA.
- **Feature flags via the database, not env vars.** `show_spotify`, `show_youtube_cards`, `is_main`, `visible` — every per-page toggle lives in the row, so the same code handles every permutation.

---

## What's next

- **Collections** — group links under labeled headers ("Music", "Coding", "Socials")
- **A/B testing** for link placement
- **Custom domains** — letting users point their own domain at their slug
- **Email-gated links** (drop-your-email-to-reveal, for newsletter funnels)
