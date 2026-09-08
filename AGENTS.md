# personal-site

Yaroslav Ivchenkov's personal visit-card page — a single static HTML file, deployed to ivchenkov.tech.

## What this is (and isn't)

- A brief personal landing page, NOT a resume/CV and NOT a LinkedIn mirror. It pitches capability ("what I bring"), not a chronological job history.
- No employer or company names anywhere on the page, no dated experience timeline, no side-project mentions. This is deliberate: the page must stay accurate without edits when the owner changes jobs. Full career history lives on LinkedIn; this page links out to it instead of duplicating it.
- Location is shown only as "Remote · UTC+3" — never state the actual country. This is an intentional privacy choice, not an oversight.

## Stack

Plain HTML + CSS + vanilla JS. No framework, no build step, no `package.json`, no dependencies. Keep it that way unless the owner explicitly asks to change stacks — the whole point of this project is staying lightweight.

- `index.html` — the entire site (single page)
- `favicon.svg` — tab icon
- `.github/workflows/deploy.yml` — CI/CD

## Deployment

Every push to `main` runs `.github/workflows/deploy.yml`:

1. `aws s3 sync` uploads everything in the repo (except `.git/`, `.github/`, `README.md`, `AGENTS.md`, `CLAUDE.md`) to a Yandex Object Storage bucket (S3-compatible, endpoint `https://storage.yandexcloud.net`), which serves `ivchenkov.tech`.
2. A second explicit `aws s3 cp` re-uploads `index.html` with `--content-type "text/html; charset=utf-8"` — plain `aws s3 sync` guesses `text/html` with no charset for `.html` files, which the raw HTTP header would otherwise ship without one. If more top-level `.html` files are ever added, extend this step (or loop over `*.html`) rather than relying on the default guess.

Required repo secrets (Settings → Secrets and variables → Actions): `YC_ACCESS_KEY_ID`, `YC_SECRET_ACCESS_KEY`, `YC_BUCKET`. These belong to a Yandex Cloud service account scoped to the bucket — see git history around the workflow's introduction for the exact console steps if they ever need recreating.

There is no staging environment and no build step to verify locally beyond opening `index.html` in a browser — changes go live on the next push to `main`.

## Design system

CSS custom properties are defined once in `:root`, redefined under `@media (prefers-color-scheme: dark)` — reuse these tokens for anything new, don't hardcode colors:

- `--bg`, `--surface`, `--line` — backgrounds and borders
- `--ink`, `--ink-dim` — body text (full-weight vs. secondary)
- `--amber` / `--amber-txt`, `--teal` / `--teal-txt`, `--violet-txt` — accent colors. The bare (`--amber`, `--teal`) variants are only safe on large text (≈19px+) or decorative elements (SVG dots, gradient strip) — they fail WCAG AA contrast at small sizes. The `-txt` variants are darkened specifically for small text (labels, card copy, links) and must be used there instead.
- Fonts: IBM Plex Mono (headings, labels, anything code/CLI-flavored) + IBM Plex Sans (body), loaded via a Google Fonts `@import`.
- Content column is capped at `max-width:1040px` (`.wrap`) — don't let prose stretch full-width on wide monitors, but don't over-narrow it either; this was tuned more than once based on real-monitor feedback, not a first guess.

## Copy conventions

- Card labels (the small mono "kind" text, e.g. "Operator", "Security") can use technical jargon — the audience includes engineers who recognize it. Card *descriptions* underneath must be plain English a non-technical reader (recruiter/HR) can understand. Don't let both the label and the description be jargon-only.
- Frame everything as capability/value ("what I bring"), never past-tense CV bullets ("I did X at Y").
- Avoid repeating the "X, not Y" negation construction more than once or twice on the page — it reads as a recognizable LLM-copy tic, flagged in an independent review of this page. Prefer positive framing.
- The `status: Ready · uptime: 9y` line and the `kubectl get engineer ...` footer line are stylized as literal CLI/terminal output — never translate or "improve" these into prose, in either language.
- Kubernetes/cloud-native terms (Kubernetes, CRD, Helm, Cluster API/CAPI, Admission control, cert names like CKA/AWS CCP) are used in English even in the Russian copy — that's genuinely how Russian-speaking engineers say them; don't transliterate or invent Russian equivalents.

## i18n (EN/RU)

The page has a language switcher (top-right, fixed position) and no i18n library — it's a small hand-rolled system in the last `<script>` block:

- Every translatable element has a `data-i18n="key"` attribute; its English text in the HTML markup also serves as the no-JS fallback.
- The `I18N` object in the script maps each key to `{en: "...", ru: "..."}`. Values may contain inline HTML (e.g. the `more_pointer` key embeds an `<a>` tag) — they're applied via `.innerHTML`, not `.textContent`.
- `detectLang()` picks a language: a stored `localStorage['site-lang']` override wins, otherwise `navigator.language` (`ru*` → Russian, else English).
- To add a new translatable string: add `data-i18n="new_key"` to the element (English text stays in the markup as-is), then add `new_key: {en:"...", ru:"..."}` to the `I18N` object. Nothing else needs to change — `applyLang()` walks all `[data-i18n]` elements generically.
- Known limitation, accepted as-is: `<meta>`/OG tags in `<head>` are static and only ever show the English copy, because link-preview crawlers (Slack, LinkedIn, etc.) don't execute JS. Don't try to "fix" this without discussing a real architecture change (e.g. separate URLs per language) first — it's a deliberate tradeoff for a single-file lightweight site, not an oversight.

## Contact email protection

The footer's email link is assembled by a small inline script at runtime (string concatenation of a user part and a host part), not present as a plain scrapable string in the HTML source. It also uses a Gmail "+" alias distinct from the owner's bare address (see the script in `index.html` for the exact value), so if it ever gets spammed, the owner can filter/block that exact alias without touching his real inbox. Keep both of these if the email is ever touched — don't "simplify" it back into a plain `mailto:` in the markup.

## Verifying changes before shipping

There's no test suite (nothing to test — it's a static page). Before pushing a visual or content change, actually look at it rendered:

- Use an isolated/headless browser session for checks against this site. If using `chrome-devtools-axi`, do not set `CHROME_DEVTOOLS_AXI_AUTO_CONNECT` — that env var attaches to the operator's real, already-open Chrome instead of a disposable one, which is almost never what you want for routine verification of this repo.
- Check at least two realistic viewports: a phone width (~390px) and a real desktop width (1440px+) — most layout/contrast issues here only showed up at one of the two, not both.
- Check computed contrast for any new/changed text color against its actual background — several past passes shipped text that failed WCAG AA until a follow-up review caught it.
- Check the browser console for errors.
- If a change touches copy, screenshot both EN and RU (click the language switcher) — don't assume symmetry between the two.
