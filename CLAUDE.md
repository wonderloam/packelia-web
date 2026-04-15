# CLAUDE.md

## Project Overview

`packelia-web` is the static GitHub Pages site served at `https://packelia.wonderloam.com`. Its sole purpose is to support the Packelia mobile app's universal links and app links — it is **not** a web app.

Two responsibilities:

1. **Host verification files** at `/.well-known/` so iOS and Android can verify the Packelia app owns this domain and route matching URLs into the app.
2. **Serve branded fallback pages** when someone taps a Packelia link (magic sign-in, trip invite) on a device without the app installed, or inside an in-app browser that blocks universal links.

The mobile app lives in the sibling `packelia` repo.

## Architecture

Pure static site, deployed via GitHub Pages (Jekyll build from `main` branch). No build step, no framework, no JS dependencies. Files are served as-is.

### URL surface

Universal links are catch-all: every path on `packelia.wonderloam.com` is registered to open the app. The two paths that actually get used today:

- `/auth/callback?code=...` — Supabase magic link redirect (set in `src/hooks/useAuth.ts` in the app repo)
- `/invite/{token}` — trip invite shares (set in `src/components/sharing/ShareBottomSheet.tsx` in the app repo)

When the app isn't installed (or an in-app browser blocks the universal link), each path maps to a fallback HTML page:

- `/auth/callback` → static file at `auth/callback/index.html`
- `/invite/{token}` → no static file exists for the dynamic token, so GitHub Pages serves `404.html` (HTTP 404 but with branded HTML body)
- `/` → `index.html` landing page

`404.html` is a smart catch-all — a small inline script reads `window.location.pathname` and swaps visible content between invite / auth / default sections.

### Continue in Packelia button

Each fallback page shows a `.continue-button` that, when tapped, opens `packelia://auth/callback?code=...` or `packelia://invite/{token}`. This is the **recovery hatch for in-app browsers** (Gmail IAB, Outlook, Slack) that block universal links. A user-initiated tap on a custom-scheme link is allowed in almost all WKWebViews, even when the OS-level universal link interception was blocked.

Styles for the button are in `assets/styles.css` as `.continue-button` and `.divider`. Both are `display: none` by default; an inline script adds `.visible` class only when the URL actually has the needed param (so the button doesn't appear for direct navigation without a code or token).

## Key files

| File | Responsibility |
|---|---|
| `CNAME` | Binds the GitHub Pages site to `packelia.wonderloam.com` |
| `_config.yml` | Jekyll: `include: [".well-known"]` to un-ignore the dotfile dir |
| `.well-known/apple-app-site-association` | iOS AASA — catch-all `"/": "/*"` for team `9B7ZA5S6XM` / bundle `com.wonderloam.packelia` |
| `.well-known/apple-app-site-association.json` | Mirror with `.json` extension — GitHub Pages serves this with correct `Content-Type: application/json`, protects against MIME issues on the canonical filename |
| `.well-known/assetlinks.json` | Android digital asset links — needs real SHA-256 fingerprint from `eas credentials --platform android` |
| `index.html` | Landing page |
| `auth/callback/index.html` | Magic link fallback with Continue button |
| `404.html` | Catch-all with path-based content switching and Continue button for `/invite/{token}` and `/auth/callback` |
| `privacy.html` | Privacy policy (required by App Store and Google Play) |
| `robots.txt` | Disallow crawlers from `/auth/` and `/invite/` |
| `assets/styles.css` | All page styles, shared across pages |

## Brand

The site mirrors the mobile app's aesthetic:

- Background `#FBF8F4` (warm off-white)
- Primary `#5C3D2E` (warm brown)
- Muted text `#9A8B7D`
- `DM Serif Display` for headings (imported via Google Fonts)
- `Outfit` for body text
- 12–20px border radii, soft shadows
- Calm, confident, premium — never loud or flashy

## Gotchas

**AASA MIME type.** Apple wants `Content-Type: application/json` for `apple-app-site-association`, but GitHub Pages serves extension-less files as `application/octet-stream`. Modern iOS clients and Apple's AASA CDN parse the content anyway, but we ship the `.json`-suffixed mirror as defensive coverage. If AASA verification still fails after deployment, the real fix is switching hosts to something with custom header support (Cloudflare Pages, Netlify).

**Don't auto-redirect on page load.** The Continue button is user-initiated on purpose. Auto-redirecting to `packelia://...` on page load:
- Fails silently in most in-app browsers (they block programmatic custom-scheme navigation)
- Shows an ugly "Cannot open URL" error if the app isn't installed
- Creates weird double-fire behavior when universal links worked but the page loaded first

The user taps a button, or they don't. No JS redirects.

**Code is not consumed by this page.** The PKCE `code` parameter in magic links is single-use, but the fallback page never calls Supabase — it just forwards the param to the app via the Continue button's `href`. This means email security scanners (Outlook ATP, Mimecast) can safely pre-fetch the URL without invalidating the code. Do not add any code that calls `exchangeCodeForSession` from the web page.

**PKCE is single-device.** Supabase magic links use PKCE with `code_verifier` stored on the device that *requested* the link. A magic link clicked on a different device (e.g., on desktop when phone requested) cannot complete sign-in regardless of whether universal links work — the static page is the correct final state in that scenario.

**Placeholders that must be replaced before production:**

- `REPLACE_WITH_SHA256_FINGERPRINT_FROM_EAS_CREDENTIALS` in `.well-known/assetlinks.json`
- `id_APPSTORE_ID` in `index.html`, `auth/callback/index.html`, `404.html` (3 files × multiple occurrences each)
- `Last updated: ...` date placeholder in `privacy.html`
- `app-screenshot.png` — referenced but not yet created; `index.html` has a TODO comment where it would render

**Jekyll quirks.** Files in `.well-known/` would be ignored by default (Jekyll ignores dotfile dirs). The `include:` directive in `_config.yml` overrides this. If we ever need Jekyll plugins beyond the GitHub Pages allowlist, switch to `.nojekyll` and lose the allowlist, or use a GitHub Action to build.

## Verifying deployment

After deploying, these commands should return real content:

```bash
curl https://packelia.wonderloam.com/.well-known/apple-app-site-association
curl https://packelia.wonderloam.com/.well-known/assetlinks.json
curl https://packelia.wonderloam.com/
curl -I https://packelia.wonderloam.com/invite/test-token   # HTTP 404 but serves 404.html body
```

Apple's AASA CDN (takes up to 24h to propagate):

```bash
curl https://app-site-association.cdn-apple.com/a/v1/packelia.wonderloam.com
```

Google digital asset links verifier:

```bash
curl "https://digitalassetlinks.googleapis.com/v1/statements:list?source.web.site=https://packelia.wonderloam.com&relation=delegate_permission/common.handle_all_urls"
```

## Relationship to the mobile app

Changes here **must** stay in sync with the app's `app.json`:

- `ios.associatedDomains` must contain `applinks:packelia.wonderloam.com`
- `android.intentFilters` must include host `packelia.wonderloam.com` with `autoVerify: true`
- `ios.appleTeamId` and the AASA `appIDs` team prefix must match
- `android.package` and `assetlinks.json` `package_name` must match

If you change the AASA components or add a new path type, no app config change is needed (the `/*` catch-all covers everything). If you change the domain, `app.json` needs updating too — breaking that link breaks magic link sign-in for all users.

## What NOT to add here

- Backend code — this is static. If you need dynamic behavior, the app handles it.
- Auth logic — the fallback page is intentionally inert. Don't call Supabase from here.
- Analytics beacons — shared invite links should not silently call trackers; respect user privacy and the install-as-landing-page positioning.
- Frameworks — Jekyll is already more machinery than this site needs. Don't add React, Next.js, etc.
