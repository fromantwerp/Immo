# Region Listings — iPhone PWA

An installable app version of the map viewer. No App Store, no Mac, no fees.

## Files
- `index.html` — the app
- `manifest.json` — tells iOS this is installable, sets the icon/name
- `service-worker.js` — caches the app so it opens instantly / works offline
- `icons/` — app icon (192px & 512px)
- (you add) `data.json` — your scraped listings, dropped in next to `index.html`

## Step 1 — Get it online (free)
iPhone requires a real URL (https) to install a PWA properly — it won't work by just
double-clicking the file on your Mac and can't be installed straight from `file://`.
Easiest free option: **GitHub Pages**.

1. Create a free GitHub account if you don't have one.
2. Create a new repository, e.g. `region-listings`.
3. Upload all the files from this folder (`index.html`, `manifest.json`,
   `service-worker.js`, `icons/`) to the repo — plus a `data.json` (the output of
   `immoweb_scraper.py --out data.json`, renamed to exactly `data.json`).
4. In the repo: **Settings → Pages → Deploy from branch → main → / (root)** → Save.
5. GitHub gives you a URL like `https://yourname.github.io/region-listings/`.

(Other free static hosts work identically: Netlify, Vercel, Cloudflare Pages — drag-and-drop the folder.)

## Step 2 — Install it on your iPhone
1. Open that URL in **Safari** (must be Safari, not Chrome — Chrome on iOS can't install PWAs).
2. Tap the **Share** icon → **Add to Home Screen**.
3. It now sits on your home screen with its own icon, opens full-screen with no browser
   bar, and stays cached for offline use.

## Updating the data
Every time you re-run the scraper:
1. `python immoweb_scraper.py --postal-codes ... --out data.json`
2. Upload/replace `data.json` in your GitHub repo (or hosting folder).
3. Reopen the app on your phone — it auto-fetches the fresh `data.json` on launch.
   (If you're offline it'll fall back to the last cached version.)

You can also skip `data.json` entirely and just tap **"Load JSON"** inside the app to
pick a file manually each time — works even without re-deploying.

## Notes
- This won't appear in App Store search or be sent to anyone else — it's just on your
  phone, installed straight from your own URL. That's normal and fine for personal use.
- If you ever want a "real" native app (App Store listing, push notifications, etc.),
  the SwiftUI route from before is the next step up — but it needs a Mac + Xcode, and
  $99/year only if you want to publish it publicly.
