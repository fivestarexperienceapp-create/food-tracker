# Mise — cook, log, know

A mobile-first PWA for cooking, meal-prep planning and honest nutrition tracking. Recipes carry a 0–100 healthiness score with an additive-risk breakdown, a cost per serving and a 1–5 difficulty rating; photos of real meals are analysed by Gemini vision using your own Google AI Studio key.

Everything runs client-side. There is no backend, no account and no analytics — the log, the week plan and the API key live in your browser's local storage.

---

## Project structure

```
Mise.dc.html      the whole app (markup + logic), the file you edit
support.js        component runtime required by Mise.dc.html
image-slot.js     image tile: prefilled from Unsplash, drag-and-drop to replace
config.js         YOUR API KEYS — gitignored, never commit
config.example.js template to copy when setting up a fresh clone
photos.json       Unsplash search results captured at build time (reference copy)
index.html        single-file build for deployment (generated — see below)
manifest.json     PWA manifest: name, icons, shortcuts, standalone display
sw.js             service worker: app-shell precache, offline navigation
icons/            192 / 512 / maskable-512 PNG app icons
README.md
```

## Keys and `config.js`

```js
window.MISE_CONFIG = {
  geminiKey: 'AIza…',      // https://aistudio.google.com/apikey
  unsplashKey: '…'         // Unsplash app Access Key
};
```

`config.js` is in `.gitignore`; `config.example.js` is the template to copy on a fresh clone. The Gemini key is seeded into Settings on first load, and you can still paste a different one there — a key entered by hand wins and is remembered.

**This is client-side code.** Anything in `config.js` is visible to anyone who opens the page, and the bundled `index.html` contains it too. Keep the repo private, use throwaway keys, or move the calls behind a serverless proxy before sharing the URL. GitHub's secret scanning will revoke a key pushed to a public repo.

## Run it locally

A service worker and the manifest both need a real HTTP origin, so serve the folder rather than opening the file directly:

```bash
python3 -m http.server 8080     # or: npx serve .
```

Then open `http://localhost:8080/index.html` (the built app) or `http://localhost:8080/Mise.dc.html` (the editable source).

## Connect Gemini

1. Get a free key at <https://aistudio.google.com/apikey>.
2. Put it in `config.js` as `geminiKey`, or open the app, tap the gear in the header and paste it there. Choose **3.6 Flash** (fast, cheap) or **3.6 Pro** (more accurate on messy plates).
3. The key is written to `localStorage` under `mise.v1` and sent from the device straight to `generativelanguage.googleapis.com`. It never passes through any server of ours.

The app uses two calls, both with a strict JSON response schema:

- **Vision** — the photo is downscaled to 1024 px and posted as inline JPEG data. Gemini returns the dish name, confidence, portion estimate, per-portion macros, a 0–100 health score with letter grade, a component list with additive risk, and one concrete improvement.
- **Recommendations** — the Discover feed's "What should I cook?" panel sends your recent log and plan density and gets three suggestions back. Without a key this one falls back to the highest-scoring dishes in the local library.

> A key pasted into a static site is visible to anyone using that browser profile. Fine for personal use; for a shared deployment, proxy the calls through a small serverless function and keep the key server-side.

## Photography (Unsplash)

Each of the twelve dishes was matched to an Unsplash photo through the Search API, and the results — image URL, photographer, profile link — are cached in the `PHOTOS` table inside `Mise.dc.html` (with a reference copy in `photos.json`). **The running app makes no Unsplash calls**, so it is immune to the 50-requests-per-hour demo limit and works offline once cached.

**Settings → Photography → New photos from Unsplash** re-queries the API live using `unsplashKey`, one search per dish (12 of your 50 hourly requests), picking a random result from the top six so you can shuffle until you like the set. Choices persist in local storage; **Reset** returns to the baked-in set. A 403 from Unsplash is reported as a rate-limit message rather than failing silently.

Every photo carries the attribution Unsplash requires ("Photo by … on Unsplash", linked to the photographer), rendered by the image tile itself — that is why dish headers sit at the top of their images, leaving the bottom-left corner clear. Drag any image file onto a tile to override it with your own shot.

## How costs work

Two numbers, because they answer different questions:

- **Food used** — the recipe's share of each ingredient (one tablespoon of harissa, not the jar). This is what divides into cost per serving.
- **Full shop** — every pack at checkout, for someone starting with an empty pantry. Much higher, and correctly so: the oil, tahini and spices carry over into dozens of later meals.

Measurements and units are US throughout: oz / lb / cups / °F, Calories rather than kcal, miles for store distance, and US ingredient names (scallions, cilantro, heavy cream, all-purpose flour).

## How the score works

`score = nutrition × 0.65 + additives × 0.25 + whole-food share × 0.1`

- **Nutrition** — penalties for energy density, saturated fat, sugar and sodium, weighted 3×; a smaller credit for fibre and protein that cannot cancel them out. A rich braise lands in the 20s, a vegetable-and-pulse dish in the 90s.
- **Additives** — every processed component costs points by risk band (low 8, moderate 20, high 34), and the offending additive is named in the recipe's ingredient list.
- **Whole food** — the proportion of ingredients that are unprocessed.

Grades: **A** 80+, **B** 65+, **C** 50+, **D** 35+, **E** below.

## Deploy

### GitHub Pages

```bash
git init
git add .
git commit -m "Mise: cooking and nutrition PWA"
git branch -M main
git remote add origin https://github.com/<you>/<repo>.git
git push -u origin main
```

Check `git status` before the first commit and confirm `config.js` is **not** listed. Deploying from a public repo means either committing `config.js` (don't) or entering the Gemini key by hand in Settings on each device.

Then **Settings → Pages → Source: Deploy from a branch → main / (root)**. The app lands at `https://<you>.github.io/<repo>/`.

Every path in the app is relative (`./sw.js`, `./manifest.json`, `./icons/...`), so it works from a subpath with no configuration. If you want the root URL to open the app, keep `index.html` at the repository root — Pages serves it automatically.

### Netlify or Vercel

Drag the folder onto Netlify, or:

```bash
npx netlify deploy --prod --dir .
npx vercel --prod
```

No build step and no framework — both treat it as a static site. Leave the build command blank and the publish directory as the project root.

### After each deploy

Bump `CACHE` in `sw.js` (`mise-v1` → `mise-v2`). The old cache is deleted on activation and clients pick up the new shell on their next visit.

## Install on a phone

Open the deployed URL, then **Share → Add to Home Screen** (iOS) or the install prompt in the address bar (Android). It launches standalone in portrait, keeps working offline for anything already cached, and the manifest shortcuts jump straight to Scan, Log or Plan.

## Dish photography

Every card and recipe header is an `<image-slot>` prefilled with its Unsplash photo. Drag your own image onto one and it replaces the photo permanently for that device.

## Editing the app

`Mise.dc.html` holds the template and the logic class together.

- **Recipes** — the `RAW` array in the logic class. Each entry needs per-serving nutrition, `cost`, `difficulty` (1–5), `prep`/`cook` minutes, `cats` (`trending`, `once`, `prep`), an `ing` list where every item carries a risk band, and `steps`. Add a matching entry to `PHOTOS` and `UNSPLASH_Q` to give a new dish photography.
- **Costs** — every ingredient carries `c` (what this recipe's share of it costs) and `p` (the price of the smallest pack you can buy). A recipe's ingredient total is `Σ c`, cost per serving is that divided by `servings`, and the recipe sheet also shows `Σ p` — the from-scratch shop. The week's shopping list totals the *unique* packs, so one jar of harissa is counted once however many dishes use it. Currency is a component prop (`$` by default).
- **Sources** — the method text is original, so no recipe carries a citation by default. Add `source: { name, url }` to a recipe and the sheet renders "Adapted from <name>" as a link above the action buttons.
- **Shops** — the `STORES` array. `mi` is the walking distance shown on the card. `keys` are the ingredient keywords that route a shopping-list item to that shop; `Directions` links open Google Maps, using your coordinates once you grant location.
- **Scoring** — `scoreRecipe()` and the `GRADES` table.
- **Tweakable settings** — calorie target, score prominence, additive notes, currency and accent colour are exposed as component props.
