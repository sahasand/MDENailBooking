# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Single-file static marketing site for "Laura Nails" — in-home manicures/pedicures in Medellín. Bilingual: Spanish-default with an English toggle. **WhatsApp is the only contact channel** — no email, no contact form, no phone display, no social links anywhere on the site. Booking happens via WhatsApp deep links with language-aware pre-filled messages. There is no backend, no auth, no package manager, no build step.

Tracked files at the root:

- `index.html` — the entire live site (HTML + inline CSS in `<style>` + inline JS in `<script>`). "Pick your polish" design: a saturated color field, sign-painter headline, photos cropped to nail shapes, and a five-shade picker that repaints the whole page.
- `zine.html` — the previous live design (maximalist "Tropical Sunset" art-zine: collage hero, marquees, spinning stamp). Kept as a backup for comparison and rollback.
- `classic.html` — the earliest restrained editorial version, also kept as a backup.
- The three designs share no CSS, fonts, or class names. Don't propagate edits between them unless asked.
- `og-image.jpg` — 1200×630 social link-preview card. Referenced by `<meta property="og:image">`. Still shows the older look.
- `laura-nails.webp`, `laura-abtme.webp`, `img1.webp` — photos used by `index.html` and `zine.html`. WebP at quality ~82; if you replace one, keep WebP and re-export at the same quality.
- `service-gel.jpg`, `service-manicure.jpg`, `service-pedicure.jpg` — used only by `classic.html`'s services section.

`.remember/` (Claude Code memory), `py.log`/`py.err.log` (artifacts from the optional `python -m http.server`), and `img1.png` (uncompressed source for `img1.webp`) may exist locally but are not part of the site — safe to ignore.

## Commands

No `package.json`, no test runner, no linter, no build pipeline. To work on the site:

- **View locally:** open `index.html` directly in a browser (file://) — images load because they sit beside it. All three HTML files work via `file://`.
- **Optional local server** (only needed if you want HTTP semantics): `python -m http.server 8910` from the repo root.
- **Deploy:** GitHub Pages serves the repo automatically; whatever lives at `index.html` is the live page. To swap which design is live, rename the files.
- **Screenshots:** headless Chrome on macOS will not go narrower than 500px, so test phone widths by loading the page in a 390px-wide `<iframe>`. Headless also does not tick main-thread transitions, so the FAQ looks stuck closed there; check it in a real browser.

## Architecture (live `index.html`)

Section order: nav → hero (headline, nails, text + CTAs, color picker) → facts strip → services (price list, add-ons, pricing notes, group block) → about → how-it-works → faq → closing CTA → footer. Pricing lives inside the services section; there is no separate pricing section.

Recurring patterns worth understanding before editing:

- **Shades drive every color.** `<html data-shade="…">` selects one of five shades named after Medellín fruit and sky: `guayaba` (default), `lulo`, `aguacate`, `cielo`, `mora`. Each shade defines six tokens at the top of the stylesheet: `--paint` (the polish color), `--on-paint` (text/buttons on it), `--deep` (body text and dark blocks), `--tint` (page background), `--wash` (alternate section background), `--shadow` (the offset drop shade). Nothing else in the CSS hard-codes a color, apart from the swatch buttons' inline `--sw` values, which must match each shade's `--paint`. To add or retune a shade, edit the token block, the swatch button, and the regex in the `<head>` script. `--on-paint` on `--paint` and `--deep` on `--tint`/`--wash` must stay at 4.5:1 or better.
- **Background contexts.** `.on-paint`, `.on-deep`, and `.on-wash` set a section's background and reassign the context tokens (`--btn-bg`, `--btn-fg`, `--ring`, `--rule`, `--soft`), so buttons, focus rings, hairlines, and secondary text adapt without per-section overrides.
- **The picker.** `.swatch` buttons in the hero call `paintFrom()`, which swaps `data-shade` inside `document.startViewTransition` and animates a circular clip from the tapped swatch, so the new color spreads across the page like polish. Falls back to an instant swap when the API is missing or reduced motion is on. The choice persists to `localStorage('shade')`, and a tiny script in `<head>` restores it before first paint. `<meta name="theme-color">` follows the shade. The shade names are site colors only; nothing claims Laura stocks polish in them, and the WhatsApp prefill does not mention them.
- **The nail shape** is one `border-radius` value, `--nail-radius` (almond tip, shallow cuticle curve), applied by `.nail`. It frames the hero and about photos, the picker swatches, and the step numbers. `.nail::after` is the gloss streak.
- **Two typefaces**, both from Google Fonts: **Shrikhand** (single weight; headline, section titles, prices, wordmark, signature) and **Asap** (variable weight and width; everything else). The hero and closing headlines carry a hard offset `text-shadow` in `--shadow`, borrowed from hand-painted salon window lettering. No monospace, no script face.
- **Icons** are two inline SVG `<symbol>`s (`#i-wa`, `#i-check`) at the top of `<body>`, referenced with `<use>`.
- **i18n is attribute-driven, with Spanish as the unconditional default.** A single `I18N` object in the inline `<script>` holds both `es` and `en` dictionaries with flat dotted keys (`hero.title`, `svc-1.name`, `faq.q1`, etc.). HTML elements opt in via three attributes:
  - `data-i18n="key"` — replaces `textContent`.
  - `data-i18n-aria="key"` — sets `aria-label`.
  - `data-i18n-alt="key"` — sets `alt` on `<img>` elements.
  - `setLang(lang)` also rewrites `document.title`, `<meta name="description">`, the `<html lang>` attribute, and the `aria-pressed` state of the `[ES][EN]` toggle. The choice persists to `localStorage('lang')`. The init logic does NOT respect `navigator.language` — first-time visitors always land on Spanish (this was an explicit user requirement).
- **Hero headline sizing is per language.** Each of the two headline spans is its own block with `text-wrap: balance`. English runs longer, so `html[lang="en"] .hero h1` has its own font sizes. Re-check both languages at 320, 390, and desktop widths after changing headline copy.
- **WhatsApp links are dynamic and language-aware.** The `WA_URL` constant in the inline `<script>` holds the bare number (`https://wa.me/573006159715` — Laura's real number). Every CTA carries `data-wa` plus a plain `href` to the same number as a no-JS fallback. An element can carry `data-wa-text-key="some.i18n.key"` to pre-fill a message in the active language; without it, the fallback is `wa.default-text`. `setLang()` rebuilds every `[data-wa]` href. To change the number, edit `WA_URL` and the fallback `href`s.
- **Hero layout.** A CSS grid with named areas. On phones the order is headline → nails → text and CTAs → picker; `.hero-side` (nails + picker) uses `display: contents` there. From 940px it becomes the right-hand column. The three nails overlap on a 12-column grid: a plain glossy nail, Laura's photo, the nail-art close-up.
- **Motion.** One load moment: the three hero nails reveal from cuticle to tip (`paint-on`, clip-path). Everything else answers an action (repaint, FAQ open, button hover). There are no scroll-reveal animations and no marquees; don't add them back. `prefers-reduced-motion` disables all of it.
- **FAQ accordion** uses a `grid-template-rows: 0fr → 1fr` transition (not `max-height`) so any answer length animates without magic numbers. Each question is a real `<button>` with `aria-expanded`; JS toggles `.is-open` on the `.faq-item`. The collapsed state only applies under `html.js`, so answers stay readable without JavaScript.
- **Services are a price list**, not cards: `.menu-item` rows laid out with grid areas (name, desc, time, price, book). The combo row is `.is-featured` and takes the paint color. Each "Reservar / Book" link uses `aria-labelledby` to read as "Reservar Manicure Clásico".
- **Mobile sticky WhatsApp button (`#wa-fab`)** sits after `</footer>`, is hidden from 768px up, and gets `.is-visible` once `window.scrollY > 600`.
- **Hero photo framing** (don't shuffle without intent): `laura-nails.webp` is anchored `center top` so her face and hands stay in frame; `img1.webp` sits at `42% center`. The about section uses `laura-abtme.webp` only.

## Copy & voice

The brand voice is Laura speaking — a paisa from Medellín, casual and warm. Voice rules from explicit user feedback that must hold for any future edits:

- **No em dashes (—) anywhere on the rendered site, in either language.** Use commas, periods, "since," or "but" instead. (En dashes also avoided.) When translating EN content into ES (or vice versa), watch for em dashes that sneak in via translation — they're a common AI tell.
- **Be honest about capabilities.** Spanish is Laura's native language; English is in-progress with Google Translate as a helper. The site mentions this explicitly in the FAQ ("Google Translate y yo somos un buen equipo") and the facts strip ("ES + EN con ayudita de Translate"). Don't reintroduce confident bilingual claims.
- **No fabricated metrics.** No "+500 clients", no "4.9★ rating", no "since 2018". Laura's actual story is "paisa, 2 años haciendo uñas, friends + neighbors + tourists". The facts strip under the hero is honest: `100% a domicilio`, `45 a 75 min por servicio`, `ES + EN con ayudita de Translate`.
- **Avoid AI-sounding marketing patterns** in both languages: parallel triplets, "designed/curated/elevated," "comfort/experience/elegance" buzzwords, hedge words like "simply/effortlessly."
- **Spanish copy uses paisa "vos" voice** (Medellín local), not Spain or Mexico Spanish. Verb forms: `escribime`, `decime`, `contame`, `querés`, `podés`, `tenés`, `salgás`, `sentás`. Drop in occasional locally-coded words where natural: `paisa`, `tranqui`, `platica`, `datáfono`, `cédula`, `portería`, `supercomún`. Do not over-slang.
- **No decorative dingbats, emoji, eyebrow labels, arrows on links, or middle-dot separators** in the live design. Spanish prices and amounts use a period for thousands (`$45.000`, `4.000 COP`); English uses a comma.

Translatable strings live as keys in the `I18N` object inside the inline `<script>`. There is no separate strings file or CMS. Always update both `es` and `en` entries when adding or editing copy.

## What's NOT on the site (intentionally)

These were considered and explicitly removed/declined — don't add them back without asking:

- **No portfolio/gallery section.** `img1.webp` is one nail-art photo used as a single small nail in the hero, not a gallery. The user has explicitly declined adding a multi-photo gallery until there are 4-5+ real shots of finished nails.
- **No testimonials section.** Same reason — no real customer quotes to ship.
- **No Instagram links** anywhere (nav, footer, mobile FAB area all stripped).
- **No email link.** WhatsApp is the sole contact.
- **No phone-number display.** WhatsApp link only — visitors click through to chat, never see the phone number on the page.
- **No fabricated trust metrics** (client counts, star ratings, year-since claims). Honest factoids only.
- **No service-area map.** Coverage is stated in text (Poblado, Laureles, Envigado).

## Open loops

- `og-image.jpg` still shows the old art-zine look. It needs a new 1200×630 card to match the live design.
- The FAQ says Laura does simple nail art only, but both hero photos show elaborate art. Swap in a photo of her own work when one exists.
- WhatsApp number changed to +57 300 6159715 on 2026-11-12, in all three HTML files.
