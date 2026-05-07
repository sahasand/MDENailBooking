# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Single-file static marketing site for "Laura Nails" — in-home manicures/pedicures in Medellín. Bilingual: Spanish-default with an English toggle. **WhatsApp is the only contact channel** — no email, no contact form, no phone, no social links anywhere on the site. Booking happens via WhatsApp deep links with language-aware pre-filled messages. There is no backend, no auth, no package manager, no build step.

Six tracked files at the root:

- `index.html` — the entire site (HTML + inline CSS in `<style>` + inline JS in `<script>`)
- `laura-nails.webp`, `laura-abtme.webp`, `service-gel.jpg`, `service-manicure.jpg`, `service-pedicure.jpg` — referenced from `index.html` by relative filename. The two `laura-*` photos are WebP for size; if you replace them, keep WebP and re-export at quality ~82.

`.remember/` (Claude Code memory) and `py.log`/`py.err.log` (artifacts from the optional `python -m http.server` below) may exist locally but are not part of the site — safe to ignore or delete.

## Commands

No `package.json`, no test runner, no linter, no build pipeline. To work on the site:

- **View locally:** open `index.html` directly in a browser (file://) — images load because they sit beside it.
- **Optional local server** (only needed if you want HTTP semantics): `python -m http.server 4000` from the repo root, then open `http://localhost:4000/`. Any static server works.
- **Deploy:** upload the six files to any static host (Netlify drop, Cloudflare Pages, GitHub Pages, S3, plain SFTP).

## Architecture

The whole page is composed in `index.html` in this section order: nav → hero → services → about → how-it-works → pricing → faq → footer-cta. Section transitions are thin gradient strips (`.strip.cream-white`, `.strip.white-cream`, `.strip.white-blush`, `.strip.blush-cream`) between each `<section>`.

A handful of patterns recur throughout the file and are worth understanding before editing:

- **Design tokens are CSS custom properties** declared on `:root` in the inline `<style>`: `--cream`, `--blush`, `--dusty-rose`, `--deep-rose`, `--espresso`, `--warm-gray`. Every color reference uses `var(--…)` — change a token, the whole site shifts. Fonts: Playfair Display (`.display` class) and Inter (body), loaded via Google Fonts `<link>` in `<head>`.
- **All icons are inline SVG** copied from lucide. No icon library, no sprite — paste a new SVG path inline at the call site and let `currentColor` carry the color.
- **i18n is attribute-driven, with Spanish as the unconditional default.** A single `I18N` object in the inline `<script>` holds both `es` and `en` dictionaries. HTML elements opt into translation via three attributes:
  - `data-i18n="key"` — replaces `textContent`. If the value contains `\n`, it's split and joined with `<br>` (used for h1/h2 line breaks). When an element has children that should NOT be replaced (e.g. a CTA with an icon `<svg>` and a text label), put the label inside its own `<span data-i18n="…">`.
  - `data-i18n-aria="key"` — sets `aria-label` (used on icon-only buttons and the modal sheet).
  - `data-i18n-alt="key"` — sets `alt` on `<img>` elements.
  - `setLang(lang)` also rewrites `document.title`, `<meta name="description">`, the `<html lang>` attribute, and the `aria-pressed` state of the `[ES][EN]` toggle pill in the nav. The choice persists to `localStorage('lang')`. The init logic does NOT respect `navigator.language` — first-time visitors always land on Spanish (this was an explicit user requirement).
- **WhatsApp links are dynamic and language-aware.** The `WA_URL` constant in the inline `<script>` holds the bare number (`https://wa.me/573016913136` — Laura's real number). Every CTA gets `data-wa` (no `href`). Optionally, an element can carry `data-wa-text-key="some.i18n.key"` to pre-fill a WhatsApp message in the active language; without that attribute, the fallback is `wa.default-text` (a generic "I saw your website" greeting). `setLang()` rebuilds every `[data-wa]` href with the right URL-encoded prefill, so toggling EN ↔ ES updates the prefill text on every CTA. To change the number, edit `WA_URL` in one place.
- **Mobile sticky WhatsApp FAB (`#wa-fab`)** sits at the end of `<body>`, before the inline `<script>`. It uses the same `data-wa` system, is hidden on desktop (`@media (min-width: 768px) { display: none; }`), and JS toggles a `.visible` class on it once `window.scrollY > 600` so it fades in only after the hero is out of view. Don't move it inside `<main>` — it must remain `position: fixed` outside the document flow.
- **Scroll-in animations use IntersectionObserver, not GSAP.** Elements get `class="reveal"` (and optionally `delay-1`…`delay-4` for staggered children); the IO adds `.in` once they hit ~15% visibility, which fires the opacity/translate transition. The `prefers-reduced-motion` media query at the bottom of the stylesheet kills all animations and the smooth-scroll behavior.
- **FAQ accordion uses a `grid-template-rows: 0fr → 1fr` transition** — not `max-height`. This is deliberate: it animates to natural content height with no magic numbers, so any answer length renders without clipping. Don't replace it with a fixed `max-height` cap. The click handler also auto-scrolls the opened question into view on mobile (`window.innerWidth < 768`); `.faq-q` has its own `scroll-margin-top` so the question header doesn't end up tucked under the fixed nav.
- **Mobile menu is an `aria-modal="true"` sheet** that slides in from the right. Open/close toggles `.open` on `#menu`, locks body scroll, and `Escape` closes it. The `[ES][EN]` language toggle sits between the desktop nav links and the hamburger so it's always visible — on mobile it sits next to the hamburger, on desktop it sits next to the WhatsApp button.
- **Section anchors use `scroll-margin-top: calc(var(--nav-h) + 8px)`** so jumping to `#faq` from the nav doesn't tuck the heading under the fixed bar.

### Section-specific patterns

- **Hero flow is claim → proof → action.** Inside `.hero-content` the children are ordered: eyebrow, h1, lede, `.trust-row` (the three pills), `.hero-cta-row`. Reveal delays follow that reading order (delay-1 through delay-4). The primary CTA inside `.hero-cta-row` is the rose `.btn-rose .btn-lg` pill; the secondary "See Services / Ver servicios" is a deliberately demoted `.hero-link-secondary` (text + chevron, underlined warm-gray). Don't promote the secondary back to a full-size outline button — the visual hierarchy is intentional.
- **Pricing has per-row "Reservar / Book" buttons (`.book-btn`)** with `data-wa-text-key` pointing to a service-specific prefill in the i18n dictionary (e.g. `pricing.basics-1-text`). The combo row uses `.price-row.featured` with a `.featured-tag` "Más pedido / Most popular" pill. USD conversion text renders inline as `<span class="price-usd">` after each COP amount; the conversion rate is hardcoded around 4,000 COP/USD with a footnote in the `.pricing-meta` strip — update both together if it shifts materially. The right column ("add-ons") uses `.pricing-col.panel` modifier — a subtle white panel on the blush background — to balance visual weight with the left column.
- **About paragraphs and chips are constrained to `max-width: 540px`** for comfortable line lengths on desktop. The chips grid collapses from `1fr 1fr` to `1fr` (single column) at `max-width: 480px`.

## Copy & voice

The brand voice is Laura speaking — a paisa from Medellín, casual and warm. Voice rules from explicit user feedback that must hold for any future edits:

- **No em dashes (—) anywhere on the rendered site, in either language.** Use commas, periods, "since," or "but" instead. (En dashes also avoided.) When translating EN content into ES (or vice versa), watch for em dashes that sneak in via translation — they're a common AI tell.
- **Avoid AI-sounding marketing patterns** in both languages: parallel triplets, "designed/curated/elevated," "comfort/experience/elegance" buzzwords, hedge words like "simply/effortlessly."
- **Spanish copy uses paisa "vos" voice** (Medellín local), not Spain or Mexico Spanish. Verb forms: `escribime`, `decime`, `contame`, `querés`, `podés`, `tenés`, `salgás`, `sentás`. Drop in occasional locally-coded words where natural: `paisa`, `tranqui`, `platica`, `datáfono`, `cédula`, `portería`, `supercomún`. Do not over-slang.

Translatable strings live as keys in the `I18N` object inside the inline `<script>`. There is no separate strings file or CMS. Always update both `es` and `en` entries when adding or editing copy.

## What's NOT on the site (intentionally)

These were considered and explicitly removed/declined — don't add them back without asking:

- **No portfolio/gallery section** (was added as Tier 1 but removed because there are no real photos of Laura's work yet).
- **No testimonials section** (same reason — no real customer quotes to ship).
- **No Instagram links** anywhere (nav, mobile sheet, gallery footer all stripped).
- **No email link** in the footer. WhatsApp is the sole contact.
- **No tertiary accent color** — palette is intentionally tight (cream/blush/rose/espresso/warm-gray).
- **No service area map.**
