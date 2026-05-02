# Deploying the marketing site

The current site at [leanlog.ai](https://leanlog.ai) is served by **GitHub Pages** off `main`. The refresh you're about to ship lives on the `refresh` branch and is meant to deploy to **Cloudflare Pages** as a preview before we flip the apex domain off GitHub.

This doc walks through (1) standing up the new Cloudflare Pages project so you can preview the refresh on a `.pages.dev` URL, and (2) the eventual cutover to `leanlog.ai`.

There is a closely related — but separate — doc at [../demos/naked-wines/DEPLOYMENT.md](../demos/naked-wines/DEPLOYMENT.md) that documents how the Naked Wines demo (`nakedwines.leanlog.ai`) is hosted. Same Cloudflare account, same patterns, but the marketing site has no auth gate, no Pages Functions, and is publicly indexable.

---

## Current state (as of refresh push)

- **Phase 1 is done.** The Pages project `leanlog-landing` exists in Oscar's Cloudflare account.
- **Production branch is `main`** (not `refresh` — see the trade-off note in Phase 1 below). `leanlog-landing.pages.dev` therefore mirrors what's on GitHub Pages today.
- **`refresh` is iterated as a preview** at `https://refresh.leanlog-landing.pages.dev`. Cloudflare auto-rebuilds on every push to `refresh`.
- **No custom domain** is attached to the Pages project yet. `leanlog.ai` still resolves to GitHub Pages.

Skip to Phase 2 if you're doing the cutover.

---

## Phase 1 — Stand up the Cloudflare Pages project (preview only)

Goal: have a working `<project>.pages.dev` URL serving the new homepage and `/about` from the `refresh` branch. The apex `leanlog.ai` is **not** touched in this phase — it keeps serving the old GitHub Pages site.

You'll do this in the Cloudflare dashboard. The whole thing takes about 5 minutes.

### 1. Push `refresh` to GitHub (once)

```bash
cd /c/dev/LeanLog/landing
git push -u origin refresh
```

(If you've already done this, skip.)

### 2. Create the Pages project

1. Sign in to Cloudflare Dashboard at https://dash.cloudflare.com using `oscar@leanlog.ai`. (Same account that hosts `demos-naked-wines`.)
2. Left sidebar: **Workers & Pages**.
3. Click **Create** (top right) → **Pages** tab → **Connect to Git**.
4. **Authorize GitHub** if you haven't (one-time, account-level).
5. Pick the repo: **`LeanLog/landing`**. Click **Begin setup**.

### 3. Configure the build

| Field | Value |
|---|---|
| Project name | `leanlog-landing` (this becomes `leanlog-landing.pages.dev`) |
| Production branch | **`main`** *(see trade-off below)* |
| Framework preset | **None** |
| Build command | *(leave empty)* |
| Build output directory | `/` *(repo root — the HTML files are at the top level)* |
| Root directory (advanced) | *(leave empty)* |
| Environment variables | *(none)* |

**Do not** set up custom domains here. Stay on the default `.pages.dev` URL until we're ready to cut over.

**Do not** copy the auth middleware or `noindex` settings from the demos project. The marketing site is public and crawlable.

Click **Save and Deploy**.

> **Trade-off on production branch.** With production=`main`, the Pages project mirrors what's on GitHub Pages today and we iterate `refresh` at `refresh.leanlog-landing.pages.dev`. With production=`refresh`, the bare `.pages.dev` URL would serve the new content. We picked `main` because (a) the eventual cutover is just a `refresh` → `main` merge, no Cloudflare branch switch needed, and (b) preview deployments get `x-robots-tag: noindex` by default — handy while the WIP isn't ready for crawlers.

### 4. Wait for the first build (~30 seconds)

Cloudflare clones the repo, copies static files, and serves them. There's no `npm run build` step — it's pure static HTML.

When it finishes, you'll see a URL like `https://leanlog-landing.pages.dev` (production = `main`) and `https://refresh.leanlog-landing.pages.dev` (preview = `refresh`). Open the one matching the branch you want to verify.

### 5. Smoke test

In a fresh incognito window:

- Homepage at `/` — verify hero cycles (Run/Manage/Handle More Clients/Suppliers/Carriers), demo tabs auto-cycle, problem section, vision rows (talk/log/control), closing CTA "Try LeanLog **now.**", footer.
- About at `/about.html` (or `/about` — Pages handles both) — verify hero copy, credentials strip, closing CTA.
- All 7 logos load (UChicago, Stanford, Georgia Tech, Amazon, J&J, Intel, MITRE).
- Mobile responsive: open DevTools (F12), toggle device toolbar (Ctrl+Shift+M), set width to 680px and 360px. Check the credentials grid wraps and the CTA buttons go full-width.

### 6. Iterate

Push to `refresh` → Cloudflare auto-rebuilds in ~30s. Push to a different branch and you get a separate preview URL like `https://<branch>.leanlog-landing.pages.dev`.

That's the loop until the copy is signed off.

---

## Phase 2 — Cut over `leanlog.ai` from GitHub Pages to Cloudflare Pages

Don't do this until the content is approved. The flip is reversible but it touches DNS, so it's worth being deliberate.

### 1. Merge `refresh` to `main`

```bash
cd /c/dev/LeanLog/landing
git checkout main
git merge --ff-only refresh   # or do a non-ff merge if you prefer
git push origin main
```

This makes the GitHub Pages site at `leanlog.ai` show the new content too. Some prefer to flip DNS first to avoid any inconsistency window — either order is fine since both will eventually serve the same content.

### 2. (Skip if production is already `main`)

If the project was set up with production=`main` (current state — see top of doc), Cloudflare auto-rebuilds `main` the moment the merge lands. Nothing to do here.

If production was set up as `refresh`, switch it now:
- **Settings** → **Builds & deployments** → **Production branch** → change `refresh` to `main`.
- Trigger a redeploy: **Deployments** tab → **Retry deployment** on the latest entry.

### 3. Add `leanlog.ai` as a custom domain

In `leanlog-landing` project:
- **Custom domains** tab → **Set up a custom domain** → enter `leanlog.ai`.
- Cloudflare detects the existing GitHub Pages A records (`185.199.108-111.153`) and prompts you to **replace** them. Confirm.
- Cloudflare handles CNAME flattening at apex automatically because the zone is on the same account.

(Optionally repeat for `www.leanlog.ai`. Today it CNAMEs to `rodrigo-ll.github.io`. You can either point it at the new project or set up a Page Rule that 301-redirects `www` to apex.)

### 4. Verify

In incognito: `https://leanlog.ai` and `https://www.leanlog.ai`. Hard-refresh (Ctrl+Shift+R) to bypass cache.

If you see a cert warning right after the swap, give it 1–5 min for Cloudflare to provision the cert. It almost always resolves on its own.

### 5. (Optional) Decommission GitHub Pages

In `LeanLog/landing` → **Settings** → **Pages** → set source to **None**. Keep this for a day or two before disabling, just in case.

The `landing/CNAME` file becomes a no-op. Safe to delete or leave.

---

## Critical guardrails

- **Do not touch MX, SPF, DMARC, or DKIM records.** Only A records change, and only at cutover. Email breaks if you fat-finger the wrong row.
- **Do not copy the auth middleware** (`functions/_middleware.js`) from the demos project. Marketing site is public.
- **Do not include `noindex` meta or `Disallow: /` robots.txt.** Marketing site should be crawlable by search engines.
- **SSL/TLS mode at the leanlog.ai zone is `Full (strict)`** and should stay that way. Switching to Flexible would cause an infinite-redirect loop on the apex.

---

## DNS state reference

The `leanlog.ai` zone is registered with **GoDaddy** (Rodrigo's account) but DNS is delegated to **Cloudflare** (`oscar@leanlog.ai` account). All DNS edits happen in Cloudflare.

Today's records (until we flip):
- `A @ → 185.199.108-111.153` (GitHub Pages)
- `CNAME www → rodrigo-ll.github.io` (GH Pages origin)
- `CNAME nakedwines → demos-naked-wines.pages.dev`
- `MX × 5` (Google Workspace) — **do not touch**
- SPF / DMARC — **do not touch**

After the flip, `A @` records change to Cloudflare's pointers and `www` either points at the new Pages project or 301s to apex.

---

## Rolling back

If something goes wrong after the cutover:

1. In `leanlog-landing` → **Custom domains** → remove `leanlog.ai`.
2. In Cloudflare DNS → restore the GitHub Pages A records (108–111.153 for the `185.199.x.x` range).
3. Re-enable GitHub Pages in the repo settings.

Total roll-back time: ~5 minutes plus DNS propagation.
