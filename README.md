# ad-funnel

An agent skill that scrapes the **complete funnel behind an ad**: the ad
creative, the destination URL, the landing page, the cart/checkout flow,
upsells, payment methods and the tracking stack.

Built for AI coding agents (opencode / Claude Code / any agent that loads
`SKILL.md` skills). Give it an ad — it gives you the whole path a click takes.

## What it does

```
ad creative  →  destination URL  →  landing page  →  cart  →  checkout  →  upsells
     │                │                  │            │          │            │
  text, CTA,      redirect chain      offer, price   platform   payment     thank-you /
  media, ID       + cloaking check     proof, urgency  + APIs    methods     OTO pages
                                    + tracking pixels          + COD?      (200 vs 404)
```

1. **Phase 1 — Get the ad.** Meta Ad Library blocks plain HTTP clients
   (curl → 403). The skill falls back to public Ad Library mirrors for the
   full ad text + destination, then uses a real browser only when needed.
2. **Phase 2 — Resolve the destination.** Full redirect chain via
   `curl -w url_effective`, with redirector/cloaking detection.
3. **Phase 3 — Scrape the landing page.** Store fingerprint (Shopify/Woo/
   funnel builder), offer details, urgency devices, COD vs prepaid hints, and
   the tracking stack (Meta Pixel, TikTok, GA4, Klaviyo…).
4. **Phase 4 — Walk the funnel.** Read-only probes of real endpoints:
   `/products/<handle>.js`, `/cart.js`, `/checkout` (detects Shopify →
   `shop.app` hosted checkout), plus existence checks for thank-you/upsell
   pages. **Never completes a purchase.**
5. **Phase 5 — Report.** One dense Markdown report delivered in the chat —
   no files created.

## Install

### opencode

```bash
git clone https://github.com/0xhzsa/ad-funnel-skill.git
# then either symlink or copy the skill folder:
cp -r ad-funnel-skill ~/.config/opencode/skills/ad-funnel
```

Restart opencode. It shows up automatically (skills are scanned from
`~/.config/opencode/skills/**/SKILL.md`).

### Claude Code / Claude-compatible agents

```bash
cp -r ad-funnel-skill ~/.claude/skills/ad-funnel
```

### Manual registration (opencode, custom path)

Add to `opencode.json`:

```json
{ "skills": { "paths": ["./path/to/ad-funnel-skill"] } }
```

## Usage

Just paste an input and trigger it naturally:

```
scrape the funnel of this ad: https://www.facebook.com/ads/library/?id=1488820192751693
```

```
what's behind this landing page? https://example-store.com
```

```
ad funnel for this tweet: https://x.com/someone/status/123...
```

Accepted inputs: Meta Ad Library links (`?id=` or search URLs), any ad
destination / landing page URL, or a social post containing the ad.

## Example output (real run)

> **🎯 Ad** — Bare Ritual · Meta Ad Library ID `1488820192751693` · ran 205
> days with 8 relaunches · Facebook, Instagram, Audience Network, Messenger,
> Threads · CTA "Shop now" → `bare-ritual.com`
> *"Most men are paying twice for skincare that doesn't work… buy one jar, get
> a second jar completely free… 90-day money-back guarantee. 470,000 jars
> sold."*
>
> **🔗 Redirect chain** — 0 redirects → `https://bare-ritual.com/`
>
> **🌍 Landing** — Shopify · "Beef Tallow Honey Balm" · PayPal + Shop Pay ·
> countdown, reviews, 90-day guarantee · tracking: Klaviyo ×10,
> **Meta Pixel absent** (paid-social ad with no pixel on the offer page)
>
> **💰 Funnel** — Landing → `/cart.js` (AUD, empty session cart) →
> `/products/tallow-balm.js` = **86.00 AUD**, 3 variants → `/checkout` **302 →
> `shop.app` universal checkout (Shop Pay enabled)** → thank-you/upsell pages
> **404 (absent)** — post-purchase funnel not built.
>
> **🧐 Red flags** — No Meta Pixel despite running paid social; no post-purchase upsell step.

## Safety rules baked into the skill

- **Probe, never purchase.** No real payments, no personal data submitted.
  Cart/checkout endpoints are read-only GETs.
- **Chat-only output.** No deliverable files are written; scratch is deleted.
- **Quote, don't guess.** Anything unretrievable is listed under
  *"⚠️ Not retrieved"* instead of being invented.

## Notes

- Written and tested on Windows (PowerShell 5.1): `curl.exe --globoff`,
  manual DNS via `8.8.8.8`, quote-handling workarounds are documented in the
  skill.
- Public ad data only (Meta Ad Library is a public transparency tool).
  Ad creative belongs to its advertisers — use it for research/competitive
  analysis, don't republish creatives as your own.

## License

MIT
