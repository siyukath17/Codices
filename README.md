# codices

> a daily oracle for programmers, in the aesthetic of Aubrey Beardsley × Oscar Wilde × Unix terminal × Co-Star astrology.

Static, single-page React prototype. No build step, no backend. Open `Codices.html` in a browser and the entire experience runs client-side.

---

## Quick start

### Online (recommended)

Serve the folder over HTTP and open it in a modern browser:

```bash
# Python 3
python3 -m http.server 8000

# or Node
npx serve .
```

Then visit <http://localhost:8000/Codices.html>.

### Deploy

The whole project is a static bundle. Drop it into any static host:

- **Vercel** — `vercel deploy` from the project root. The included `Codices.html`, `cards.json`, and `assets/` get served as-is. No config needed; no build step. Works perfectly.
- **Netlify / GitHub Pages / Cloudflare Pages** — same story, drag-and-drop the folder.
- **Any S3 / Nginx / Apache** — point them at the directory.

`Codices.html` is the entry point. If you want a friendlier root URL, rename it to `index.html` or add a redirect at your host.

### Opening from `file://` directly

You can double-click `Codices.html` and it will load, but with one caveat — see the troubleshooting section below.

---

## Project layout

```
.
├── Codices.html        ← the entire app (HTML + inline CSS + inline React/Babel)
├── cards.json          ← all 22 card definitions (also embedded inline as fallback)
├── assets/             ← Beardsley-style illustrations, one per card
│   ├── 00-fool.png
│   ├── 01-engineer.png
│   └── ...
└── README.md
```

That's the whole project. There's no `node_modules`, no bundler config, no transpile step. React, ReactDOM, and Babel are pinned via `<script>` tags from `unpkg.com`. `html-to-image` and `qrcode-generator` come from `jsdelivr` for the share card.

---

## How the deck works

### Drawing a card

`drawCardForUser(birthDigits)` is deterministic. Same inputs always produce the same card.

1. User enters birth date on the landing screen as `YYYYMMDD` (auto-formatted from digits).
2. Today's date in `YYYY-MM-DD` is fetched from `localStorage` cache first — same birth + same day returns instantly.
3. Otherwise the seed is composed: `"birthDigits | today | julianDate | moonSeed | timezone"`.
4. The seed is hashed with SHA-256 (Web Crypto API). The first 8 hex chars become an integer; `mod 22` selects the card.
5. The full draw object (cardIndex, seedHex, moonPhase, moonIllumination, julianDate, timezone) is cached under `card_{birthDigits}_{today}` in `localStorage`.

Moon phase comes from `api.farmsense.net/v1/moonphases/?d=<unix>` (cached per day under `moon_<date>`). If the request fails, the algorithm degrades gracefully to `{Phase: "Unknown", Illumination: 0.5}` so the same card still draws every time.

The console logs the full draw object — open DevTools to debug.

### Cards as data

All 22 cards live in `cards.json` with this schema:

```json
{
  "id": "00",
  "roman": "0",
  "name_traditional": "the fool",
  "name_codex": "// hello world",
  "name_technical": "fool.init()",
  "illustration_path": "assets/00-fool.png",
  "reading": { "opening": "…", "today": "…", "body": "…" },
  "do_list":   ["…", "…"],
  "dont_list": ["…", "…"],
  "lucky": {
    "color":     "#FF6B35",
    "commit":    "feat: initial",
    "time":      "14:00 – 16:00",
    "avoid":     "code review meetings",
    "companion": "XVII the star"
  }
}
```

The file is loaded two ways for resilience:

- **Inline copy** — a `<script type="application/json" id="cards-data">` block in the HTML head. Always works, including under `file://`.
- **Fetched copy** — `fetch("cards.json")`. Runs on top of the inline copy so updates to the JSON show up without rebuilding the HTML. Silently no-ops if the request fails (which happens under `file://`).

If you edit `cards.json`, the inline copy in `Codices.html` will be stale. Re-embed it with:

```bash
# crude one-liner: pipe cards.json into the script tag in-place
node -e '
  const fs = require("fs");
  const html = fs.readFileSync("Codices.html", "utf8");
  const json = fs.readFileSync("cards.json", "utf8");
  const re = /<script type="application\/json" id="cards-data">[\s\S]*?<\/script>/;
  const tag = "<script type=\"application/json\" id=\"cards-data\">" + json + "</script>";
  fs.writeFileSync("Codices.html", html.replace(re, tag));
'
```

…or just deploy and rely on the fetched copy.

### Dev mode

Press **`D`** anywhere outside the input to toggle a dropdown in the top-right corner that cycles all 22 cards. It re-mounts the reading screen so each card replays its reveal sequence with its own content.

---

## The screens

| # | name | what happens |
|---|---|---|
| 1 | **Landing** | brand top-left, italic lede, mono prompt, hairline input with a red blinking cursor, three dim footnotes bottom-left. Enter a valid date → screen 2 |
| 2 | **Compile + Circuit Oracle** | left column types out a 9-line log at 30cps; right column animates a 22-node gold circuit diamond in lockstep. Real values (timezone, moon, julian date, seed) are templated into the log once the draw resolves. |
| 3 | **Card reveal** | staged 4-second entrance of the chosen card: scale-in → gold double frame draws → roman → traditional name → illustration → divider → italic title → mono subtitle. L-bracket + flower frame corners fade in around the viewport. |
| 4 | **Reading** | scroll-linked. Card morphs left + scales down + rotates -3°; flowers fade out; right column reveals 4 fade-in segments: prose / DO / DON'T / LUCKY block (with color swatch). `share →` opens a 1080×1620 PNG export modal. |

Tap the card on screen 3 or 4 to flip to the back — a shared deck-pattern: gold double frame, tapestry-style ✦ bands top + bottom, edge flowers, and a heraldic motif in the center. No personal data on the back.

---

## Sharing

Clicking `share →` on the reading screen opens a modal with a 1080×1620 image:

- Double gold frame with ✦ corner ornaments
- Top + bottom tapestry bands
- 600×840 illustration
- Roman, codex title, technical subtitle
- Quoted opening line
- Today's date
- `◇ codices` + 160×160 gold-framed QR pointing to `https://codices.app`
- `scan to explore` caption

`↓ download` renders the modal contents as a PNG via `html-to-image` and triggers a browser download.

---

## Troubleshooting

### "I downloaded the project. Only The Fool shows up. Dev picker doesn't show the other 21 cards."

If you open `Codices.html` directly via `file://` (e.g. double-clicking it), your browser refuses to `fetch("cards.json")` for security reasons. The app falls back to the inline `<script id="cards-data">` block — which contains all 22 cards now — so this should be fixed.

If you're still seeing only The Fool, your local copy of `Codices.html` was bundled with a stale inline JSON. Either:

1. **Run a local server** (one-liner above) — `fetch` will succeed and pull the live `cards.json`.
2. **Re-embed** the inline JSON using the snippet in the "Cards as data" section.

### Will it work on Vercel?

Yes, straightforwardly. Vercel serves over HTTPS, `fetch("cards.json")` works, all 22 cards load. No build config needed. Just drop the folder in and deploy.

### "The moon phase shows `…` forever."

`api.farmsense.net` is occasionally slow or down. The app waits up to ~5s, then falls back to `Illumination: 0.5` for the seed and the line stays as `…`. The draw still completes deterministically. If you want to remove the network dependency entirely, replace `fetchMoonPhase()` in `Codices.html` with a stub that returns the fallback.

### "The download button gives me a tiny / weird image."

`html-to-image` occasionally races with web font loading. Wait 1-2 seconds after the modal opens before clicking download so Cormorant Garamond + JetBrains Mono are fully painted.

### "The card back looks black in PowerPoint / Keynote on import."

Don't import the PNG. The modal IS the export — it's already a flat raster. If you need a vector version, that doesn't exist yet.

---

## License

Concept, copy, and visual system © codices. Beardsley illustrations are derivative works of public-domain originals.
# Codices
# Codices
# Codices
