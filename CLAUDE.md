# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Single-file static marketing site for "Laura Nails" — in-home manicures/pedicures in Medellín. Bilingual: Spanish-default with an English toggle. **WhatsApp is the only contact channel** — no email, no contact form, no phone display, no social links anywhere on the site. Booking happens via WhatsApp deep links with language-aware pre-filled messages. There is no backend, no auth, no package manager, no build step.

Tracked files at the root:

- `index.html` — the entire live site (HTML + inline CSS in `<style>` + inline JS in `<script>`). Maximalist "Tropical Sunset" art-zine aesthetic with photo collage hero, dual marquees, spinning stamp, and hand-drawn accents.
- `classic.html` — earlier restrained editorial version, kept as a backup at `/classic.html` for comparison and rollback. Completely different design system (different fonts, palette, section order, class prefix). Don't propagate edits between the two unless asked.
- `og-image.jpg` — 1200×630 social link-preview card. Referenced by `<meta property="og:image">` in both files.
- `laura-nails.webp`, `laura-abtme.webp`, `img1.webp` — photos used by `index.html`. WebP at quality ~82; if you replace one, keep WebP and re-export at the same quality.
- `service-gel.jpg`, `service-manicure.jpg`, `service-pedicure.jpg` — used only by `classic.html`'s services section.

`.remember/` (Claude Code memory), `py.log`/`py.err.log` (artifacts from the optional `python -m http.server`), and `img1.png` (uncompressed source for `img1.webp`) may exist locally but are not part of the site — safe to ignore.

## Commands

No `package.json`, no test runner, no linter, no build pipeline. To work on the site:

- **View locally:** open `index.html` directly in a browser (file://) — images load because they sit beside it. Both `index.html` and `classic.html` work via `file://`.
- **Optional local server** (only needed if you want HTTP semantics): `python -m http.server 8910` from the repo root. Visit `http://127.0.0.1:8910/` for the live design or `/classic.html` for the backup.
- **Deploy:** GitHub Pages serves the repo automatically; whatever lives at `index.html` is the live page. To swap which design is live, rename `index.html ↔ classic.html`.

## Architecture (live `index.html`)

Section order: nav → hero → marquee → marquee.reverse → services (with add-ons + pricing notes + group-rate card) → about → how-it-works → faq → cta → footer. Pricing is integrated into the services section (per-card prices + add-ons + payment / coverage / USD-conversion notes + group-rate CTA), there is no separate pricing section.

The site uses a `.d3-*` class prefix throughout (a vestige from when this design was prototyped as a React component). All visual selectors are scoped under `.d3-root`.

Recurring patterns worth understanding before editing:

- **Design tokens are CSS custom properties** declared on `.d3-root` in the inline `<style>`: `--cream`, `--peach`, `--coral`, `--terracotta`, `--plum`, `--gold`, `--olive`, `--ink`. Every color reference uses `var(--…)` — change a token, the whole site shifts.
- **Four typefaces**, each load-bearing for a specific role, all from Google Fonts:
  - **Fraunces** (variable serif) — display headlines (`.d3-display`), service prices, step numbers, FAQ questions
  - **Caveat** (handwritten cursive) — title accents, scribbles, eyebrows, signature, group-rate eyebrow
  - **Outfit** (geometric sans) — body text. Replaced Inter to avoid the generic "AI-default" font feel.
  - **JetBrains Mono** — small labels, sticker text, stat labels, pricing-note labels
- **All icons are inline SVG** (e.g., the WhatsApp icon path is duplicated inside the nav CTA, hero CTA, FAB, and CTA section button). No icon library, no sprite — paste a new SVG path inline at the call site and let `currentColor` carry the color.
- **i18n is attribute-driven, with Spanish as the unconditional default.** A single `I18N` object in the inline `<script>` holds both `es` and `en` dictionaries with flat dotted keys (`hero.title`, `svc-1.name`, `faq.q1`, etc.). HTML elements opt in via three attributes:
  - `data-i18n="key"` — replaces `textContent`. If the value contains `\n`, it's split and joined with `<br>` (used for line-breaks in headings).
  - `data-i18n-aria="key"` — sets `aria-label` (used on icon-only buttons like the FAB).
  - `data-i18n-alt="key"` — sets `alt` on `<img>` elements.
  - `setLang(lang)` also rewrites `document.title`, `<meta name="description">`, the `<html lang>` attribute, and the `aria-pressed` state of the `[ES][EN]` toggle pill in the nav. The choice persists to `localStorage('lang')`. The init logic does NOT respect `navigator.language` — first-time visitors always land on Spanish (this was an explicit user requirement).
- **WhatsApp links are dynamic and language-aware.** The `WA_URL` constant in the inline `<script>` holds the bare number (`https://wa.me/573016913136` — Laura's real number). Every CTA gets `data-wa` (no `href`). Optionally, an element can carry `data-wa-text-key="some.i18n.key"` to pre-fill a WhatsApp message in the active language; without that attribute, the fallback is `wa.default-text` (a generic "I saw your website" greeting). `setLang()` rebuilds every `[data-wa]` href with the right URL-encoded prefill, so toggling EN ↔ ES updates the prefill text on every CTA. To change the number, edit `WA_URL` in one place.
- **Mobile sticky WhatsApp FAB (`#wa-fab`)** sits at the end of `<body>`, after `.d3-root` closes. It uses the same `data-wa` system, is hidden on desktop (`@media (min-width: 768px) { display: none; }`), and JS toggles a `.visible` class on it once `window.scrollY > 600` so it fades in only after the hero is out of view. Keep it outside `.d3-root` — it must remain `position: fixed` outside the document flow.
- **Scroll-in animations use IntersectionObserver.** Elements get `class="reveal"` (optionally with `delay-1`…`delay-5` for staggered children); the IO adds `.in` once they hit ~12% visibility, which fires the opacity/translate transition. The hero gets a separate page-load CSS animation stagger that runs immediately on load (eyebrow → h1 → sub → CTAs → stats), no scroll trigger needed. The `prefers-reduced-motion` media query at the end of the stylesheet disables all animations including the spinning stamp, line-drawing underline, marquees, hover transforms, and reveal observers.
- **FAQ accordion uses a `grid-template-rows: 0fr → 1fr` transition** — not `max-height`. This is deliberate: it animates to natural content height with no magic numbers, so any answer length renders without clipping. The inner `<p data-i18n="faq.aN">` has `min-height: 0; overflow: hidden` so the row collapse works. JS toggles `.open` on the parent `.d3-faq-item`; the `+` icon also rotates 45° to become `×` via CSS transform on the open state.
- **Mobile responsiveness has dedicated breakpoint overrides** for the design's elaborate elements: `@media (max-width: 900px)` shrinks the spinning stamp from 130-150px → 92px and tucks it into the corner, sizes down sticker badges, reduces the giant outlined service numbers from 200px → 130px, and tightens hero stats so the 3-column strip fits 360-414px viewports. `@media (max-width: 560px)` shrinks further and hides one of the two stickers + two of the three floating dingbats to keep the hero readable.

### Section-specific patterns

- **Hero is a photo-collage** with three rotated cards (4:5 rectangle + 1:1 circle + 1:1 square), tape decorations, a spinning circular SVG textPath stamp ("LAURA NAILS · MEDELLÍN · A DOMICILIO" with italic Fraunces "tranqui · EST 2024" centerpiece), two sticker badges with hard-shadow offsets, three floating dingbats (`✿ ✦ ✿`), a dotted-grid backdrop, and an animated hand-drawn SVG underline that draws itself on page load under the cursive headline accent. The headline uses `<span data-i18n="hero.title">` for the upright Fraunces line and `<span data-i18n="hero.title-accent">` inside `<span class="d3-h1-script">` for the Caveat cursive payoff line.
- **Hero photo identities** (don't shuffle without intent):
  - card 1 (`.p1` rect) = `laura-nails.webp` wide editorial framing
  - card 2 (`.p2` circle) = same `laura-nails.webp` source but tight-cropped to the hands+nails detail via `object-position: center 78%; transform: scale(1.15)`
  - card 3 (`.p3` square) = `img1.webp` colorful nail-art close-up
  - About section uses `laura-abtme.webp` exclusively (different studio shot) so each surface has unique imagery
- **Two opposing marquees** sit between the hero and services: a forward strip on terracotta with service words ("manicure · pedicure · gel · a domicilio · Poblado · Laureles · Envigado · Medellín") and a counter-flow strip on plum with vibe words in JetBrains Mono caps ("tranqui · sin afanes · a tu ritmo · en tu casa"). Both run on CSS keyframes (`d3marquee` / `d3marqueeRev`) and stop via `prefers-reduced-motion`.
- **Service cards live in `.d3-svc-grid` with mixed treatments**: card 1 white, card 2 `.feat` (peach→coral gradient), card 3 white, card 4 `.feat-2.wide` (plum→terracotta gradient, full-width with `grid-column: 1/-1`). Each card has a `data-num` attribute that renders as a 200px outlined italic Fraunces number in the corner via `::before { content: attr(data-num); -webkit-text-stroke: 2px ...; color: transparent }`. Each card's "Reservar / Book" button carries `data-wa data-wa-text-key="svc-N.prefill"` for a service-specific prefilled WhatsApp message.
- **Practical pricing meta** stacks below the service cards, in this order: `.d3-svc-addons` (dashed-border panel with 4 add-ons in a 2-column `+` list), `.d3-pricing-notes` (3-column strip with payment methods, travel/coverage policy, USD conversion disclaimer; collapses to 1-column at <720px), `.d3-group-card` (plum-gradient postcard with Caveat eyebrow + Fraunces serif body + gold pill CTA prefilled with the group-rate WhatsApp message).
- **About is a 2-column grid**: left = photo wrap with rotated frame + sticky-note quote bottom-right; right = eyebrow + headline (`<span data-i18n="about.title">Hola, soy</span> <em data-i18n="about.title-accent">Laura</em>` pattern) + paragraphs + 4 chips + paisa stamp (`★ Paisa · 2 años haciendo uñas ★`) + Caveat signature.
- **How-it-works has 3 steps in a 3-col grid** with rounded-circle gradient numbers and hand-drawn dashed SVG curve connectors (`.d3-step-curve`) between adjacent steps. Mobile collapses to single-column and hides the connectors.
- **CTA is a full-bleed section** with a plum→terracotta→gold gradient background, an SVG-noise grain overlay (`::after`), and centered `✿ ✦ ✿` dingbat trio above the headline. Big cream-colored pill button as the WhatsApp CTA.

## Copy & voice

The brand voice is Laura speaking — a paisa from Medellín, casual and warm. Voice rules from explicit user feedback that must hold for any future edits:

- **No em dashes (—) anywhere on the rendered site, in either language.** Use commas, periods, "since," or "but" instead. (En dashes also avoided.) When translating EN content into ES (or vice versa), watch for em dashes that sneak in via translation — they're a common AI tell.
- **Be honest about capabilities.** Spanish is Laura's native language; English is in-progress with Google Translate as a helper. The site mentions this explicitly in the FAQ ("Google Translate y yo somos un buen equipo") and the hero stat label ("ES + EN / con ayudita de Translate"). Don't reintroduce confident bilingual claims.
- **No fabricated metrics.** No "+500 clients", no "4.9★ rating", no "since 2018". Laura's actual story is "paisa, 2 años haciendo uñas, friends + neighbors + tourists". The hero stats trio is honest: `100% / a domicilio`, `45-75 min / por servicio`, `ES + EN / con ayudita de Translate`.
- **Avoid AI-sounding marketing patterns** in both languages: parallel triplets, "designed/curated/elevated," "comfort/experience/elegance" buzzwords, hedge words like "simply/effortlessly."
- **Spanish copy uses paisa "vos" voice** (Medellín local), not Spain or Mexico Spanish. Verb forms: `escribime`, `decime`, `contame`, `querés`, `podés`, `tenés`, `salgás`, `sentás`. Drop in occasional locally-coded words where natural: `paisa`, `tranqui`, `platica`, `datáfono`, `cédula`, `portería`, `supercomún`. Do not over-slang.

Translatable strings live as keys in the `I18N` object inside the inline `<script>`. There is no separate strings file or CMS. Always update both `es` and `en` entries when adding or editing copy.

## What's NOT on the site (intentionally)

These were considered and explicitly removed/declined — don't add them back without asking:

- **No portfolio/gallery section.** `img1.webp` is one nail-art photo used as a single card in the hero collage, not a gallery. The user has explicitly declined adding a multi-photo gallery until there are 4-5+ real shots of finished nails.
- **No testimonials section.** Same reason — no real customer quotes to ship.
- **No Instagram links** anywhere (nav, footer, mobile FAB area all stripped).
- **No email link.** WhatsApp is the sole contact.
- **No phone-number display.** WhatsApp link only — visitors click through to chat, never see the phone number on the page.
- **No fabricated trust metrics** (client counts, star ratings, year-since claims). Honest factoids only.
- **No service-area map.** Coverage is stated in text (Poblado, Laureles, Envigado).
