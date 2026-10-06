---
name: ad-funnel
description: Scrapes the complete funnel behind an ad - the ad creative, its destination URL, the landing page, checkout flow, upsells, payment methods and tracking stack. Use when the user pastes a Meta Ad Library link, an ad post, or a landing page URL and wants to spy on / analyze / scrape the funnel, the offer, the redirect chain, order bumps, upsells, COD vs prepaid, or competitor landing pages. Triggers: "ad funnel", "funnel scraper", "scrape this ad", "what's behind this ad", "landing page funnel", "competitor funnel", "ad spy", "Meta Ad Library".
---

# Ad Funnel Scraper

Scrape the **whole path an ad takes**: ad creative → destination URL → landing
page → cart → checkout → upsells/thank-you page → tracking stack.

**Output rule: chat only.** Deliver the final report as one Markdown message in
the conversation. Do NOT create report files, `.md` deliverables, or archives.
Temporary scratch (a payload file for curl, a temp HTML fetch) is allowed only
when strictly necessary and must be deleted before finishing.

## Inputs the skill accepts

1. **Meta Ad Library URL** — `facebook.com/ads/library/?id=<archive_id>` or a
   search URL (`&q=` brand keyword).
2. **Any ad destination / landing page URL** (direct mode).
3. **X/Twitter or other social post containing the ad** — fetch the post first
   (webfetch works on x.com status URLs) to pull the destination link out of it.

## Phase 0 - Environment checks (Windows / PowerShell 5.1)

- Resolve DNS manually if local DNS is broken:
  `nslookup <host> 8.8.8.8` then `curl.exe --resolve "<host>:443:<IP>"`.
- Always use `curl.exe` (not `curl`) with `--globoff` (PowerShell globbing
  breaks URLs with `[`/`]`), `--ssl-no-revoke`, a browser `-A` UA, and
  `--max-time 30`.
- PS 5.1 mangles quotes in native args: write any JSON/body to a temp file and
  pass `--data-binary "@<file>"`. Delete temp files when done.
- Fetch order of preference: `webfetch` → `curl.exe` (raw HTML) → browser
  (below). Only open a browser when the page is JS-rendered and the static
  fetch came back empty or blocked.

## Phase 1 - Get the ad (Meta Ad Library)

Meta's Ad Library blocks plain HTTP clients (curl → 403, webfetch → transport
error), and the page is JS-heavy. Use this fallback ladder:

1. **Mirror first (fastest, no browser).** Read the numeric ad ID from the
   Ad Library URL (`?id=`), then fetch a public mirror of the entry, e.g.
   `https://trycrush.ai/ad-library/ad/<id>` with webfetch. Mirrors expose the
   full ad text, Meta Ad Library ID, platforms, run dates, relaunch count, EU
   reach, **and the destination domain + CTA copy**. Other mirrors
   (foreplay, adscan-type scrapers) work the same way. Note in the report that
   details came from a mirror.
2. **Browser for the official source.** If the user needs the canonical
   listing or the mirror lacks the destination URL, load the `playwright-cli`
   skill and open `facebook.com/ads/library/?id=<id>` in the live browser
   (CDP). From the ad detail view extract: page name, start date, status,
   platforms; primary text, headline, description, CTA label; media (for video,
   grab the `.mp4` src from the DOM); and the real link behind the CTA (open
   "See ad details" — EU transparency data shows there too).
3. **Direct mode** (input was already a landing/post URL): just extract the
   destination URL from the post or copy.

Always record: **Meta Ad Library ID**, advertiser, run duration/relaunches if
available (longevity = proven-winner signal), and the destination host.

## Phase 2 - Resolve the destination

```powershell
curl.exe -sSL --globoff --ssl-no-revoke -A "<browser UA>" --max-time 30 `
  -o NUL -w "final=%{url_effective}`nredirects=%{num_redirects}`n" "<url>"
```

Record the **full redirect chain** (per-hop `Location:` headers via `-v` if
needed). Flag:

- Redirectors / trackers: `go.`, `bit.`, `t.`, `redir.` cloaking domains,
  `?ref=`, `subid=`, MGID/PopAds-style domains.
- **Domain change mid-chain** (ad domain → completely different TLD): classic
  dropshipping pattern — note both domains.

## Phase 3 - Scrape the landing page

Static fetch first (`curl.exe` → scratch HTML in temp → parse, or `webfetch`).
If the body is an empty SPA shell or a challenge page, render with the browser
instead.

Extract and hold in memory for the report:

1. **Store fingerprint**: Shopify (`Shopify.theme`, `/cdn/shop/`, `/products/`),
   WooCommerce, SPA/funnel builders (ClickFunnels, Funnelish, ReConvert,
   Zipify, CartHook), currency + language, store domain.
2. **Offer**: product name, price(s), compare-at price, discount/coupon code,
   bundles (BOGO), quantity discounts, shipping cost, guarantees, trial terms.
3. **Page structure**: H1/H2s, section order, reviews/testimonials (names +
   counts), urgency devices (countdown, stock, "X people viewing"), trust
   badges, FAQ.
4. **Payment posture**: COD vs prepaid hints — forms asking only phone/address
   (COD), payment logos (Visa/PayPal/Western Union/Buck/"Cash on delivery"),
   currency. Check `mentions`/`conditions` links for hidden recurring clauses
   (past funnels in this workspace used a masked "14-day recurring" line).
5. **Tracking stack**: Meta Pixel (`fbq(`, `connect.facebook.net`,
   `facebook.com/tr`), TikTok Pixel, GA4 (`G-XXXXXXX`), Google Ads
   (`gtag('aw`), Snapchat, Pinterest, Klaviyo/Mailchimp, push apps, and
   bot-detection (DataDome, Cloudflare challenge). Missing Meta Pixel on an ad
   funnel is itself worth noting.

## Phase 4 - Walk the funnel beyond the landing page

Chat-only means: **probe, don't purchase.** Never submit a real payment or
personal data.

1. **Product → cart** (read-only Shopify endpoints):
   - `GET /products/<handle>.js` → title, `price` (minor units), variants,
     availability. Handle comes from `/products/<handle>` links in the HTML.
   - `GET /products.json?limit=N` → catalog overview.
   - `GET /cart.js` → currency, item count (session-only cart).
   - Do **not** POST add-to-cart unless the user explicitly asks.
2. **Cart → checkout**: `GET /cart`, `/checkout`, `/cart/checkout`. On Shopify,
   `/checkout` answers **302 → `shop.app/checkout/...` universal checkout**
   (follow with `-o NUL -w`, don't dump the tokenized URL into the report —
   record only "Shopify checkout hosted on shop.app, Shop Pay enabled").
   Note: this GET mints a session checkout token; that is a harmless
   session-side effect, no order is created.
3. **Checkout content** (if reachable without submitting): payment methods
   offered, COD availability, order-bump checkboxes in source, upsell app
   scripts (`reconvert`, `zipify`, `oneclick`, `upsell`), pixel events
   (`InitiateCheckout`, `AddPaymentInfo`).
4. **Post-purchase pages** — existence probe only:
   `GET /thank-you`, `/thank_you`, `/checkout/thank_you`, `/pages/thanks`,
   `/upsell`, `/offer`, `/otc`. 200 vs 404 tells you whether a
   thank-you/upsell step exists; document it as "detected/absent (404)".
5. **Cloaking check**: compare `curl` HTML vs browser-rendered HTML. If the
   bot-visible page differs from what a real visitor sees (offer hidden from
   curl, innocuous page on bot IP), flag **cloaking suspected** and report
   both variants.

## Phase 5 - Report (single Markdown message, in chat)

```markdown
## 🎯 Ad
- Source: <platform + Ad Library link / post>
- Advertiser · date · status · CTA
- Primary text / headline / description (verbatim, truncate if huge)
- Media: <type + URL>
- Longevity: <days running, relaunches — proven-winner signal>

## 🔗 Redirect chain
hop 1 → hop 2 → … → final URL (`url_effective` from curl)
- Redirectors/trackers detected: …
- Domain change mid-chain: ⚠️ yes/no

## 🌍 Landing page
- Offer: product, list price / sale price, coupon, shipping, guarantees
- Structure: H1 + section order
- Proof/urgency: reviews (names), counters, badges
- Payments: COD ✅/❌ · methods shown · currency
- Tracking: Meta Pixel (id or "absent"), TikTok, GA4, gtag-ads, email/push
- Stack: <eCom platform + detected funnel apps>

## 💰 Funnel
1. Landing → 2. Cart → 3. Checkout → 4. Thank-you/Upsell (exists?)
- Order bumps / upsells detected: …
- 404 steps (absent stages): …

## 🧐 Notes / red flags
Cloaking, hidden clauses (recurring, "14 days"), domain shuffling, price
mismatch between ad and landing, missing legal pages, etc.
```

Rules:

- Quote verbatim, don't invent. If something couldn't be retrieved (403, login
  wall, JS wall), say so explicitly under a **⚠️ Not retrieved** line instead
  of guessing — and use the Phase 1 fallback ladder before giving up.
- Keep it dense — this is an analysis chat message, not an essay.
- One report per ad. If the user gives several ads, run the phases per ad and
  send one report each.
- Delete any scratch files created during the run before finishing.
