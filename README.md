# packelia-web

Static GitHub Pages site for `packelia.wonderloam.com`. Hosts:

- Universal link verification files (Apple `apple-app-site-association`, Android `assetlinks.json`)
- Branded fallback web pages shown when someone taps a Packelia link without the app installed
- A root landing page with App Store / Google Play badges
- Privacy policy (required by both app stores)

## Structure

```
packelia-web/
├── CNAME                              # GitHub Pages custom domain
├── _config.yml                        # Jekyll config (includes .well-known)
├── robots.txt                         # Disallow /auth/ and /invite/ from indexing
├── .well-known/
│   ├── apple-app-site-association       # iOS universal link verification (canonical)
│   ├── apple-app-site-association.json  # Mirror with .json extension for correct MIME type on GitHub Pages
│   └── assetlinks.json                  # Android app link verification
├── index.html                         # Landing page
├── privacy.html                       # Privacy policy
├── auth/
│   └── callback/
│       └── index.html                 # Auth callback fallback (static path)
├── 404.html                           # Smart catch-all for /invite/{token} and other dynamic paths
└── assets/
    ├── logo.png                       # 72x72 (copied from app's packelia_round.png)
    ├── favicon.png                    # 32x32 (copied from app's packelia_favicon.png)
    ├── apple-touch-icon.png           # 1024x1024 (copied; iOS will downscale to 180x180)
    ├── og-image.png                   # 1024x1024 (copied; ideal social size is 1200x630)
    ├── app-screenshot.png             # NOT YET ADDED — index.html has a TODO comment for it
    └── styles.css                     # Shared brand styles
```

## Required actions before publishing

### 1. Replace placeholder SHA-256 fingerprint

In `.well-known/assetlinks.json`, replace `REPLACE_WITH_SHA256_FINGERPRINT_FROM_EAS_CREDENTIALS` with the real Android signing key fingerprint:

```bash
cd /path/to/packelia
eas credentials --platform android
```

If using Google Play App Signing, include both the upload key and the Play signing key fingerprints as separate entries.

### 2. Replace `id_APPSTORE_ID`

Once the app is published to the App Store, replace `id_APPSTORE_ID` in:
- `index.html`
- `auth/callback/index.html`
- `404.html`

### 3. Replace placeholder image assets (recommended)

Initial assets were copied from the app's `assets/` folder so the site is functional out of the box:

| File | Current | Recommended replacement |
|------|---------|-------------------------|
| `assets/logo.png` | 72x72 (packelia_round.png) | Optionally larger 160x160 for retina |
| `assets/favicon.png` | 32x32 | Fine as-is |
| `assets/apple-touch-icon.png` | 1024x1024 | Crop/resize to 180x180 to save bandwidth |
| `assets/og-image.png` | 1024x1024 square | Replace with 1200x630 landscape for proper social card |
| `assets/app-screenshot.png` | **MISSING** | Add a real app screenshot, ~600x1200, then uncomment the `<img>` tag in `index.html` |

### 4. Review privacy policy

`privacy.html` is fully written based on what the codebase actually collects (Supabase, PostHog analytics, RevenueCat subscriptions, Open-Meteo weather, push notifications). Review it for accuracy before publishing, set the "Last updated" date, and confirm the contact email and account-deletion flow described match your actual setup.

## Setup

### 1. Create the repo

```bash
gh repo create wonderloam/packelia-web --public --source=. --push
```

### 2. DNS

Add a CNAME record for `wonderloam.com`:

```
Type:   CNAME
Name:   packelia
Value:  wonderloam.github.io
```

### 3. Enable GitHub Pages

In the repo settings → Pages:
- Source: Deploy from branch
- Branch: `main`, folder: `/ (root)`
- Wait for custom domain verification
- Enable "Enforce HTTPS"

### 4. Verify

```bash
curl https://packelia.wonderloam.com/.well-known/apple-app-site-association
curl https://packelia.wonderloam.com/.well-known/assetlinks.json
curl https://packelia.wonderloam.com/
curl -I https://packelia.wonderloam.com/invite/test-token  # should return 404 (serving 404.html content)
```

### 5. Verify with platform tools

```bash
# Apple CDN (may take up to 24h to update)
curl https://app-site-association.cdn-apple.com/a/v1/packelia.wonderloam.com

# Google digital asset links
curl "https://digitalassetlinks.googleapis.com/v1/statements:list?source.web.site=https://packelia.wonderloam.com&relation=delegate_permission/common.handle_all_urls"
```

## How it works

- iOS fetches `/.well-known/apple-app-site-association` to verify the app owns this domain. The catch-all `"/": "/*"` means every path on this domain opens the app (if installed).
- Android fetches `/.well-known/assetlinks.json` to verify the app owns this domain (via signing certificate fingerprint).
- When the app IS installed AND the link is tapped in a context that supports universal links (Apple Mail, Safari, Chrome, most system contexts), OS intercepts the URL and opens the app directly.
- When the app is NOT installed, the browser loads the matching HTML page (with App Store / Play Store links).
- When the app IS installed but the link was tapped inside an in-app browser (Gmail iOS, Outlook, Slack) that blocks universal links, the fallback page is shown with a **"Continue in Packelia"** button. This button uses the `packelia://` custom scheme — a user-initiated tap bypasses most IAB restrictions and opens the app. The button reads the `code` or invite token from the current URL and passes it through.
- GitHub Pages serves `404.html` for any path without a matching file — a client-side script detects `/invite/` vs `/auth/callback` vs other and shows the appropriate content, including a Continue button populated with the relevant parameters.

## Known gotchas

### AASA MIME type

Apple prefers `Content-Type: application/json` for `apple-app-site-association`, but GitHub Pages serves extension-less files as `application/octet-stream`. Modern iOS clients (iOS 13+) and Apple's AASA CDN parse the file correctly regardless of Content-Type, but to be defensive this site ships **both**:

- `.well-known/apple-app-site-association` — canonical filename Apple checks first
- `.well-known/apple-app-site-association.json` — mirror with `.json` extension. GitHub Pages will serve this with `Content-Type: application/json`. Apple falls back to this if the canonical version fails MIME validation.

If you still hit verification issues after deploying, switch hosting to Cloudflare Pages or Netlify, which allow custom headers and can force the correct Content-Type on the canonical filename.

### External store badge images

The App Store and Google Play badges are loaded from external URLs (`developer.apple.com` and Wikipedia). These are fragile dependencies — if those URLs change, the badges break. Safer long-term: download the official badges and host them locally in `/assets/`.

### Jekyll vs `.nojekyll`

This site uses Jekyll (because `_config.yml` is present) with `include: [".well-known"]` to un-ignore the dotfile directory. If you want to bypass Jekyll entirely, delete `_config.yml` and add a `.nojekyll` file instead. Either approach works; the current setup preserves the option to use Jekyll features later.
