# Mobile Frontend Hosting

Serve an Ionic/Angular single-page app at `/mobile` on a Frappe bench, shipped as
versioned tarballs from GitHub Releases that install during `bench build`. You end
with `https://a.example.com/mobile` serving the SPA — deep links included — where
shipping a new version is a one-line pin change plus `bench build`, and rolling back
is the same.

This guide assumes a working bench (see
[Production Deployment](02-production-deployment.md)), an Angular monorepo you control,
and a GitHub account that can create releases and fine-grained tokens. See the
[conventions and placeholders](README.md#placeholders); the sample version tags
(`v1.4.0`, and so on) are illustrative — use your real release tags.

```
┌─ <monorepo> (Angular monorepo, private) ────────────────────────┐
│  GitHub Actions: tag push → production release                  │
│                  workflow_dispatch → prerelease                 │
│  Turbo builds libs + <angular-app> → tarball on GitHub Release  │
└──────────────────────────────────────┬──────────────────────────┘
                                       │ <angular-app>-vX.Y.Z[-dev.N].tar.gz
                                       ▼
┌─ <frappe-app> (Frappe app, private) ────────────────────────────┐
│  package.json "build" → scripts/fetch_frontend.py               │
│  downloads + extracts the pinned tarball                        │
└──────────────────────────────────────┬──────────────────────────┘
                                       │ bench build
                                       ▼
┌─ Bench ─────────────────────────────────────────────────────────┐
│  .<frappe-app>/.env   (PAT + FRONTEND_VERSION pin)              │
│  public/mobile/       → /assets/<frappe-app>/mobile/*  (assets) │
│  www/mobile.html      → /mobile                         (shell) │
└──────────────────────────────────────┬──────────────────────────┘
                                       ▼
                 https://a.example.com/mobile/sign-in, …
```

The design rests on four properties. The [routine operations](#4-routine-operations)
later lean on each, so they're worth holding onto:

- **Path-based routing** under `/mobile` — deep links work via Frappe's
  `website_route_rules`.
- **Immutable releases** — rollback is a pin change, not a rebuild.
- **Opt-in per bench** — no `.env` means `bench build` skips the fetch.
- **Per-bench version pinning** — each bench fetches the version it pins.

One complication shapes the entire Angular build: the HTML shell is served at
`/mobile` while assets live under `/assets/<frappe-app>/mobile/*`. Reconciling those
two URL roots is what [the build configuration](#routing-and-build-config) handles.

The work runs in three parts, usually at different times and often by different
people — a frontend developer cuts the release, then whoever owns a bench installs
it. Each part ends by handing a concrete artifact to the next: part 1 produces a
released tarball, part 2 produces the Frappe app that fetches it, and part 3 turns both
into a live `/mobile`.

## Contents

1. [Build and release the Angular frontend](#1-build-and-release-the-angular-frontend)
2. [Add the Frappe route and fetch script](#2-add-the-frappe-route-and-fetch-script)
3. [Install and serve on the bench](#3-install-and-serve-on-the-bench)
4. [Routine operations](#4-routine-operations)
5. [Troubleshooting](#5-troubleshooting)

## 1. Build and release the Angular frontend

Everything in this part lives in `<monorepo>` on your development machine, and you do
it once per frontend release. It sets up the build, generates the PWA assets, verifies
locally, and publishes a versioned tarball to GitHub Releases — the artifact
[part 3](#3-install-and-serve-on-the-bench) fetches.

### Routing and build config

These changes set up the runtime asset base, path-based routing, and the build
(`angular.json`) so the app is hostable at `/mobile` with assets resolving against
`/assets/<frappe-app>/mobile/`.

Resolve the asset base at runtime so one bundle works for every site. Add `assetBase`
to both environment files — `environment.ts` (dev) gets `assetBase: ''` (relative
paths work under `<base href="/">`), and `environment.prod.ts` gets the absolute
asset root:

```tsx
export const environment = {
  production: true,
  // frappeUrl resolved at runtime in main.ts so one bundle works for every site.
  frappeUrl: '',
  assetBase: '/assets/<frappe-app>/mobile',
};
```

Create the injection token at `apps/<angular-app>/src/app/shared/tokens.ts`:

```tsx
import { InjectionToken } from '@angular/core';

export const ASSET_BASE = new InjectionToken<string>('ASSET_BASE');
```

Wire `main.ts` before `bootstrapApplication(...)` to resolve `frappeUrl` from the
window origin, and add the `ASSET_BASE` provider:

```tsx
import { environment } from './environments/environment';
import { ASSET_BASE } from './app/shared/tokens';

// Resolve frappeUrl from window.location.origin so HTTPS works
// even under Cloudflare Flexible (where the origin sees HTTP).
if (environment.production) {
  environment.frappeUrl = window.location.origin;
}

// in the providers array:
{
  provide: ASSET_BASE,
  useValue: environment.assetBase
},
```

> 💡 Stay on path-based routing — don't add `withHashLocation()`. The Frappe-side
> `website_route_rules` keep deep links and refreshes working.

Inject `ASSET_BASE` wherever assets are referenced rather than hardcoding
`/assets/...`. A transloco loader, for instance, requests
`${this.assetBase}/assets/i18n/${lang}.json`; a template uses
`[src]="assetBase + '/assets/<frappe-app>-logo.png'"`.

Configure the build in `apps/<angular-app>/angular.json` — this is where the two URL
roots get reconciled. `deployUrl` rewrites Webpack-emitted asset URLs to the assets
root, but it leaves hand-authored `<link>`/`<meta>` tags in `index.html` untouched, so
a relative `href` there would resolve against `<base href="/mobile/">` and 404, where
Frappe serves only the shell. That's why the production build ships a separate
`index.html` with absolute URLs ([index.html](#indexhtml)). Set `outputPath`
and `assets` in the build **options**, and `baseHref`, `deployUrl`, `fileReplacements`,
and `index` in the **production** configuration:

```json
{
  "projects": {
    "app": {
      "architect": {
        "build": {
          "options": {
            "outputPath": {
              "base": "www",
              "browser": ""
            },
            "assets": [
              {
                "glob": "manifest*.webmanifest",
                "input": "src",
                "output": ""
              },
              {
                "glob": "**/*",
                "input": "src/assets",
                "output": "assets"
              }
            ]
          },
          "configurations": {
            "production": {
              "baseHref": "/mobile/",
              "deployUrl": "/assets/<frappe-app>/mobile/",
              "fileReplacements": [
                {
                  "replace": "src/environments/environment.ts",
                  "with": "src/environments/environment.prod.ts"
                }
              ],
              "index": {
                "input": "src/index.prod.html",
                "output": "index.html"
              }
            }
          }
        }
      }
    }
  }
}
```

- `outputPath.base: "www"` — the build output directory; leave it as is.
- `outputPath.browser: ""` — makes Angular emit directly into `www/` (not
  `www/browser/`), which every later path depends on.
- `assets` — the first glob copies both `manifest*.webmanifest` files
  ([manifests](#manifests)) to the build root; the second copies everything
  under `src/assets`.
- `production.baseHref: "/mobile/"` — emitted as `<base href="/mobile/">`, which the
  root page URLs and relative links resolve against.
- `production.deployUrl` — prepends `/assets/<frappe-app>/mobile/` to Webpack-emitted
  chunk URLs (the rewrite described above).
- `production.fileReplacements` — swaps `environment.ts` for `environment.prod.ts` in
  production builds.
- `production.index` — takes `index.prod.html` ([index.html](#indexhtml)) as
  input and writes plain `index.html`, so the production build ships the absolute-path
  variant.

> 💡 Why the `index` override and not `fileReplacements`? Angular's schema accepts
> only TS/JS/JSON in `fileReplacements` — for HTML, the `index` override is the
> supported route.

Finally, confirm the Turbo `build` task in `turbo.json` builds workspace dependencies
first and declares its outputs:

```json
{
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", "www/**"]
    }
  }
}
```

### Icons and favicon

Generate every icon size the manifest, iOS, and Android expect, using PowerShell and
`System.Drawing` so there's no extra tooling on Windows. Drop a single high-res square
PNG (minimum 512×512, recommended 1024×1024+, with ~10% margin so Android's maskable
mask doesn't crop it) at `apps/<angular-app>/src/assets/<frappe-app>-logo-icon.png`,
then generate the icon set:

<details>
<summary><strong>Generate the icon set (PowerShell)</strong></summary>

```powershell
Add-Type -AssemblyName System.Drawing
$source = Resolve-Path "apps\<angular-app>\src\assets\<frappe-app>-logo-icon.png"
$outDir = "apps\<angular-app>\src\assets\icon"
New-Item -ItemType Directory -Force $outDir | Out-Null
$srcImg = [System.Drawing.Image]::FromFile($source)

function Save-Resized {
    param($size, $bgColor, $filename)
    $bmp = New-Object System.Drawing.Bitmap $size, $size
    $g = [System.Drawing.Graphics]::FromImage($bmp)
    $g.SmoothingMode = [System.Drawing.Drawing2D.SmoothingMode]::HighQuality
    $g.InterpolationMode = [System.Drawing.Drawing2D.InterpolationMode]::HighQualityBicubic
    $g.PixelOffsetMode = [System.Drawing.Drawing2D.PixelOffsetMode]::HighQuality
    $g.CompositingQuality = [System.Drawing.Drawing2D.CompositingQuality]::HighQuality
    $g.Clear($bgColor)
    $g.DrawImage($srcImg, 0, 0, $size, $size)
    $bmp.Save("$outDir\$filename", [System.Drawing.Imaging.ImageFormat]::Png)
    $g.Dispose()
    $bmp.Dispose()
}

# "any" purpose — transparent background
@(72, 96, 128, 144, 152, 192, 384, 512) | ForEach-Object {
    Save-Resized $_ ([System.Drawing.Color]::Transparent) "icon-$_.png"
}

# "maskable" purpose — white background fill so the OS mask doesn't clip transparent pixels
@(192, 512) | ForEach-Object {
    Save-Resized $_ ([System.Drawing.Color]::White) "icon-maskable-$_.png"
}

# 32x32 favicon (browser tab)
Save-Resized 32 ([System.Drawing.Color]::Transparent) "favicon.png"
$srcImg.Dispose()
```

</details>

> 💡 `any` vs `maskable`: Android masks installed icons into a circle or rounded
> square, so a maskable icon needs a solid fill behind it with the artwork inside the
> inner ~80% safe zone.

Then bundle 16/32/48 px PNGs into a multi-resolution `favicon.ico` for Windows
shortcuts and older browsers:

<details>
<summary><strong>Build favicon.ico (PowerShell)</strong></summary>

```powershell
Add-Type -AssemblyName System.Drawing
$source = Resolve-Path "apps\<angular-app>\src\assets\<frappe-app>-logo-icon.png"
$srcImg = [System.Drawing.Image]::FromFile($source)
$sizes = @(16, 32, 48)
$pngBytes = @{}

foreach ($size in $sizes) {
    $bmp = New-Object System.Drawing.Bitmap $size, $size
    $g = [System.Drawing.Graphics]::FromImage($bmp)
    $g.SmoothingMode = [System.Drawing.Drawing2D.SmoothingMode]::HighQuality
    $g.InterpolationMode = [System.Drawing.Drawing2D.InterpolationMode]::HighQualityBicubic
    $g.PixelOffsetMode = [System.Drawing.Drawing2D.PixelOffsetMode]::HighQuality
    $g.CompositingQuality = [System.Drawing.Drawing2D.CompositingQuality]::HighQuality
    $g.Clear([System.Drawing.Color]::Transparent)
    $g.DrawImage($srcImg, 0, 0, $size, $size)
    $ms = New-Object System.IO.MemoryStream
    $bmp.Save($ms, [System.Drawing.Imaging.ImageFormat]::Png)
    $pngBytes[$size] = $ms.ToArray()
    $ms.Dispose()
    $g.Dispose()
    $bmp.Dispose()
}
$srcImg.Dispose()

$out = New-Object System.IO.MemoryStream
$writer = New-Object System.IO.BinaryWriter($out)
$writer.Write([uint16]0)
$writer.Write([uint16]1)
$writer.Write([uint16]$sizes.Count)
$offset = 6 + (16 * $sizes.Count)
foreach ($size in $sizes) {
    $bytes = $pngBytes[$size]
    $writer.Write([byte]$size)
    $writer.Write([byte]$size)
    $writer.Write([byte]0)
    $writer.Write([byte]0)
    $writer.Write([uint16]1)
    $writer.Write([uint16]32)
    $writer.Write([uint32]$bytes.Length)
    $writer.Write([uint32]$offset)
    $offset += $bytes.Length
}
foreach ($size in $sizes) {
    $writer.Write($pngBytes[$size])
}
[System.IO.File]::WriteAllBytes("apps\<angular-app>\src\assets\icon\favicon.ico", $out.ToArray())
$writer.Dispose()
$out.Dispose()
```

</details>

The result is `icon-72.png … icon-512.png` (transparent), `icon-maskable-192/512.png`
(white), `favicon.png` (32×32), and `favicon.ico` (16/32/48 multi-res) under
`apps/<angular-app>/src/assets/icon/`.

### Manifests

`start_url` and `scope` differ between dev and production, so keep two manifests;
everything else is identical. The dev manifest is
`apps/<angular-app>/src/manifest.webmanifest`:

```json
{
  "name": "<App Name>",
  "short_name": "<App>",
  "description": "<App description>",
  "start_url": "/",
  "scope": "/",
  "display": "standalone",
  "orientation": "portrait",
  "background_color": "#ffffff",
  "theme_color": "#3b82f6",
  "icons": [
    {
      "src": "assets/icon/icon-72.png",
      "sizes": "72x72",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "assets/icon/icon-192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "assets/icon/icon-512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any"
    },
    {
      "src": "assets/icon/icon-maskable-192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "maskable"
    },
    {
      "src": "assets/icon/icon-maskable-512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "maskable"
    }
  ]
}
```

The prod manifest, `apps/<angular-app>/src/manifest.prod.webmanifest`, is identical
except for the two routing keys:

```json
{
  "start_url": "/mobile",
  "scope": "/mobile"
}
```

> 💡 The manifest's icon `src` paths stay relative in both variants — a browser
> resolves them against the manifest's own URL, not the page's — so only `start_url`
> and `scope` differ.

> ⚠️ In the prod manifest it's `/mobile` — **never** `/mobile/`. Frappe
> 301-redirects `/mobile/` to `/mobile`, and the W3C scope check is a literal
> `String.startsWith()`. With a trailing slash, the post-redirect URL `/mobile`
> (where users land) is out of scope, so Chrome shows the URL-bar overlay on every
> launch and breaks standalone mode.

### index.html

Keep two `index.html` variants — dev with relative paths, prod with absolute paths
under `/assets/<frappe-app>/mobile/...` (why: `deployUrl` doesn't rewrite hand-authored
tags, [routing and build config](#routing-and-build-config)). Keep everything else in
both files (head meta, body, loading styles) **identical** — only the PWA `<link>` and
`<meta>` tags differ.

The dev variant is `apps/<angular-app>/src/index.html`:

```html
<link rel="icon" type="image/x-icon" sizes="any" href="assets/icon/favicon.ico"/>
<link rel="icon" type="image/png" sizes="32x32" href="assets/icon/favicon.png"/>

<!-- PWA -->
<link rel="manifest" href="manifest.webmanifest"/>
<meta name="theme-color" content="#3b82f6"/>

<!-- iOS home screen -->
<meta name="mobile-web-app-capable" content="yes"/>
<meta name="apple-mobile-web-app-capable" content="yes"/>
<meta name="apple-mobile-web-app-status-bar-style" content="black"/>
<meta name="apple-mobile-web-app-title" content="<App Name>"/>
<link rel="apple-touch-icon" href="assets/icon/icon-192.png"/>
<link rel="apple-touch-icon" sizes="152x152" href="assets/icon/icon-152.png"/>
<link rel="apple-touch-icon" sizes="192x192" href="assets/icon/icon-192.png"/>
```

The prod variant is `apps/<angular-app>/src/index.prod.html`:

```html
<link rel="icon" type="image/x-icon" sizes="any" href="/assets/<frappe-app>/mobile/assets/icon/favicon.ico"/>
<link rel="icon" type="image/png" sizes="32x32" href="/assets/<frappe-app>/mobile/assets/icon/favicon.png"/>

<!-- PWA -->
<link rel="manifest" href="/assets/<frappe-app>/mobile/manifest.prod.webmanifest"/>
<meta name="theme-color" content="#3b82f6"/>

<!-- iOS home screen -->
<meta name="mobile-web-app-capable" content="yes"/>
<meta name="apple-mobile-web-app-capable" content="yes"/>
<meta name="apple-mobile-web-app-status-bar-style" content="black"/>
<meta name="apple-mobile-web-app-title" content="<App Name>"/>
<link rel="apple-touch-icon" href="/assets/<frappe-app>/mobile/assets/icon/icon-192.png"/>
<link rel="apple-touch-icon" sizes="152x152" href="/assets/<frappe-app>/mobile/assets/icon/icon-152.png"/>
<link rel="apple-touch-icon" sizes="192x192" href="/assets/<frappe-app>/mobile/assets/icon/icon-192.png"/>
```

Both `mobile-web-app-capable` and `apple-mobile-web-app-capable` ship (the W3C-standard
and Apple names). The explicit `apple-touch-icon` tags are needed on top of the
manifest icons because iOS Safari ignores the manifest `icons` for the home screen,
falling back to a page screenshot when these are absent.

### Build and verify locally

Verify the build before releasing. For the dev build, the output should show
`<base href="/">`, relative hrefs, `"start_url": "/"`, and `"scope": "/"`:

```bash
cd apps/<angular-app>
npx ng build --configuration development
grep -E '<base|rel="manifest"|rel="apple-touch-icon"' www/index.html
grep -E 'start_url|scope' www/manifest.webmanifest
```

For the prod build, the output should show `<base href="/mobile/">`, absolute
`/assets/<frappe-app>/mobile/...` hrefs, `"start_url": "/mobile"`, and
`"scope": "/mobile"` (**no trailing slash**):

```bash
npx ng build --configuration production
grep -E '<base|rel="manifest"|rel="apple-touch-icon"' www/index.html
grep -E 'start_url|scope' www/manifest.prod.webmanifest
```

Verifying the live deployment comes later, once the app is installed on a site
([install on sites and verify](#install-on-sites-and-verify)).

### The release workflow

A GitHub Actions workflow builds the app and publishes a versioned tarball — the
artifact the bench fetches during [install and build](#install-and-build). Add the
workflow at `.github/workflows/<angular-app>-release.yml`. Production releases fire on
a tag push; pre-releases fire on demand:

<details>
<summary><strong><code>&lt;angular-app&gt;-release.yml</code></strong></summary>

```yaml
name: <angular-app> / Release

on:
  push:
    tags:
      - '<angular-app>-v*'

  workflow_dispatch:
    inputs:
      version:
        description: 'Version label (e.g., v1.4.0, v1.4.0-dev.1, v1.4.0-rc.1)'
        required: true
        type: string

concurrency:
  group: <angular-app>-release-${{ github.event.inputs.version || github.ref_name }}
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-22.04

    permissions:
      contents: write   # required to create releases

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Determine version
        id: ver
        run: |
          if [ "${{ github.event_name }}" = "workflow_dispatch" ]; then
            VERSION="${{ inputs.version }}"
          else
            VERSION="${GITHUB_REF_NAME#<angular-app>-}"
          fi
          echo "version=$VERSION" >> "$GITHUB_OUTPUT"
          echo "Building version: $VERSION"

      - name: Validate version
        run: |
          if [[ ! "${{ steps.ver.outputs.version }}" =~ ^v[0-9]+\.[0-9]+\.[0-9]+(-[a-zA-Z0-9.-]+)?$ ]]; then
            echo "::error::Version must match vX.Y.Z or vX.Y.Z-suffix (got: ${{ steps.ver.outputs.version }})"
            exit 1
          fi

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'

      - name: Install workspace dependencies
        run: npm ci

      - name: Build (Turbo handles library deps)
        run: npx turbo run build --filter=<angular-app>

      - name: Create tarball
        run: |
          tar -czf <angular-app>-${{ steps.ver.outputs.version }}.tar.gz \
            -C apps/<angular-app>/www .
          ls -lh <angular-app>-${{ steps.ver.outputs.version }}.tar.gz

      - name: Upload tarball as workflow artifact
        uses: actions/upload-artifact@v4
        with:
          name: <angular-app>-${{ steps.ver.outputs.version }}-${{ github.sha }}
          path: <angular-app>-${{ steps.ver.outputs.version }}.tar.gz
          retention-days: 30

      - name: Create GitHub Release with asset
        uses: softprops/action-gh-release@v2
        with:
          files: <angular-app>-${{ steps.ver.outputs.version }}.tar.gz
          generate_release_notes: true
          prerelease: ${{ contains(steps.ver.outputs.version, '-') }}
          tag_name: ${{ github.event_name == 'workflow_dispatch' && format('<angular-app>-{0}', steps.ver.outputs.version) || github.ref_name }}
```

</details>

The workflow has two triggers. For a **production release**, push a tag:
`git tag <angular-app>-v1.4.0`, then `git push origin <angular-app>-v1.4.0`. For a
**staging, dev, or RC build**, go to **Actions → Run workflow** and enter a label like
`v1.4.0-dev.1` or `v1.4.0-rc.1` — any version containing a `-` is published as a
prerelease automatically.

**Part 1 is done** when the Actions run is green and the release page shows
`<angular-app>-v1.4.0.tar.gz` attached as an asset. That tarball is the handoff to
[part 3](#3-install-and-serve-on-the-bench): a bench fetches it by tag, so nothing else
from this part needs to reach the server.

## 2. Add the Frappe route and fetch script

This part lives in `<frappe-app>`. It defines the `/mobile` route, the script that
pulls the pinned tarball, and the build hook that runs it — the Frappe-side contract
that the bench fills in and executes in [part 3](#3-install-and-serve-on-the-bench).
You do it once, then commit and push.

### Route and controller

Add the controller at `<frappe-app>/<frappe-app>/www/mobile.py`. The route `/mobile`
exists because this file does; its companion `mobile.html` is written by the fetch
script at build time. `no_cache = 1` stops Frappe caching the SPA shell, so a new build
reaches users on their next request:

```python
import frappe

# Must be at module level — Frappe reads this before calling get_context().
no_cache = 1

def get_context(context):
    return context
```

The shell and assets arrive at build time, so keep them out of version control —
append to the root `.gitignore`:

```
# Frontend artifacts populated at build time by the fetch script
<frappe-app>/public/mobile/
<frappe-app>/www/mobile.html
```

Add the catch-all route in `hooks.py` so a refresh on `/mobile/sign-in` serves
`mobile.html` instead of returning 404, letting the Angular router handle the path:

```python
website_route_rules = [
    {
      "from_route": "/mobile/<path:app_path>",
      "to_route": "mobile"
    },
]
```

> 💡 Hook changes need `bench --site <site> migrate` (or `clear-cache`) to take
> effect.

### The fetch script

The script pulls the pinned tarball from GitHub Releases and installs it; every
`bench build` runs it. Create the package directory first:

```bash
mkdir -p <frappe-app>/<frappe-app>/scripts
touch <frappe-app>/<frappe-app>/scripts/__init__.py
```

Then add the script at `<frappe-app>/<frappe-app>/scripts/fetch_frontend.py`:

<details>
<summary><strong><code>fetch_frontend.py</code></strong></summary>

```python
"""
Fetch the pinned frontend tarball from <angular-app>'s GitHub Release
and install it into <frappe-app>/public/mobile/ + <frappe-app>/www/mobile.html.

Invoked by the `build` script in package.json during `bench build`.
Configuration: <bench>/.<frappe-app>/.env (mode 600).
Skips silently if not configured.
"""

from __future__ import annotations

import json
import os
import shutil
import sys
import tarfile
import tempfile
from pathlib import Path
from urllib.error import HTTPError, URLError
from urllib.parse import quote
from urllib.request import Request, urlopen

# --- Paths -----------------------------------------------------------------

SCRIPT_DIR = Path(__file__).resolve().parent
APP_PYTHON_DIR = SCRIPT_DIR.parent          # <bench>/apps/<frappe-app>/<frappe-app>/
BENCH_DIR = APP_PYTHON_DIR.parent.parent.parent  # <bench>/
PUBLIC_MOBILE_DIR = APP_PYTHON_DIR / "public" / "mobile"
WWW_MOBILE_HTML = APP_PYTHON_DIR / "www" / "mobile.html"
STAMP_FILE = PUBLIC_MOBILE_DIR / ".version"
CONFIG_FILE = BENCH_DIR / ".<frappe-app>" / ".env"

def log(msg: str) -> None:
    print(f"[fetch_frontend] {msg}", flush=True)

def read_config() -> dict | None:
    """Return config dict, or None if file is absent (caller should skip)."""
    if not CONFIG_FILE.exists():
        return None

    mode = CONFIG_FILE.stat().st_mode & 0o777
    if mode & 0o077:
        raise SystemExit(
            f"Config file {CONFIG_FILE} has mode {oct(mode)}; expected 0o600. "
            f"Run: chmod 600 {CONFIG_FILE}"
        )

    config: dict = {}
    for line in CONFIG_FILE.read_text().splitlines():
        line = line.strip()
        if not line or line.startswith("#"):
            continue
        if "=" not in line:
            continue
        key, _, value = line.partition("=")
        config[key.strip()] = value.strip().strip('"').strip("'")

    # Env vars override file values.
    for k in ("GITHUB_PAT", "GITHUB_OWNER", "GITHUB_REPO",
              "ASSET_NAME_PREFIX", "FRONTEND_VERSION"):
        if v := os.environ.get(k):
            config[k] = v

    return config

def required(config: dict, key: str) -> str:
    value = config.get(key)
    if not value:
        raise SystemExit(f"{key} not set in {CONFIG_FILE} or environment.")
    return value

def already_installed(version: str) -> bool:
    if not STAMP_FILE.exists():
        return False
    return STAMP_FILE.read_text().strip() == version

def github_api(url: str, pat: str, accept: str) -> bytes:
    req = Request(
        url,
        headers={
            "Authorization": f"Bearer {pat}",
            "Accept": accept,
            "User-Agent": "<frappe-app>-fetch-frontend",
            "X-GitHub-Api-Version": "2022-11-28",
        },
    )
    try:
        with urlopen(req, timeout=60) as resp:
            return resp.read()
    except HTTPError as e:
        body = e.read().decode("utf-8", errors="replace")
        raise SystemExit(f"GitHub API error {e.code} for {url}\n{body}")
    except URLError as e:
        raise SystemExit(f"Network error fetching {url}: {e}")

def find_tarball_asset(version: str, config: dict) -> str:
    """Return the asset API URL for the frontend tarball in the release."""
    owner = required(config, "GITHUB_OWNER")
    repo = required(config, "GITHUB_REPO")
    prefix = config.get("ASSET_NAME_PREFIX", "<angular-app>")
    pat = required(config, "GITHUB_PAT")

    tag_encoded = quote(version, safe="")
    api_url = f"https://api.github.com/repos/{owner}/{repo}/releases/tags/{tag_encoded}"

    data = json.loads(github_api(api_url, pat, "application/vnd.github+json"))

    for asset in data.get("assets", []):
        name = asset["name"]
        if name.startswith(f"{prefix}-") and name.endswith(".tar.gz"):
            return asset["url"]

    raise SystemExit(
        f"No {prefix}-*.tar.gz asset found in release {version}. "
        f"Assets present: {[a['name'] for a in data.get('assets', [])]}"
    )

def download_asset(asset_api_url: str, pat: str, dest: Path) -> None:
    """Fetch the binary asset (Accept: octet-stream returns the file via redirect)."""
    log(f"Downloading {asset_api_url}")
    data = github_api(asset_api_url, pat, "application/octet-stream")
    dest.write_bytes(data)
    log(f"Downloaded {len(data) / 1024 / 1024:.1f} MB to {dest}")

def extract_tarball(tarball: Path, target: Path) -> None:
    if target.exists():
        shutil.rmtree(target)
    target.mkdir(parents=True, exist_ok=True)

    log(f"Extracting to {target}")
    target_resolved = target.resolve()
    with tarfile.open(tarball, "r:gz") as tf:
        # Refuse absolute paths or paths escaping the target dir.
        for member in tf.getmembers():
            mpath = (target / member.name).resolve()
            if mpath != target_resolved and not mpath.is_relative_to(target_resolved):
                raise SystemExit(f"Refusing to extract unsafe path: {member.name}")
        tf.extractall(target)

    if not (target / "index.html").exists():
        raise SystemExit(f"index.html not found in extracted tarball at {target}")

def copy_index_to_www() -> None:
    src = PUBLIC_MOBILE_DIR / "index.html"
    WWW_MOBILE_HTML.parent.mkdir(parents=True, exist_ok=True)
    shutil.copy2(src, WWW_MOBILE_HTML)
    log(f"Copied {src.name} → {WWW_MOBILE_HTML}")

def main() -> None:
    """Entry point invoked by `bench build` via package.json."""
    config = read_config()
    if config is None:
        log(f"Mobile frontend not configured (no {CONFIG_FILE}). Skipping.")
        return

    version = config.get("FRONTEND_VERSION", "").strip()
    if not version:
        log(f"Mobile frontend disabled (FRONTEND_VERSION not set in {CONFIG_FILE}). Skipping.")
        return

    log(f"Pinned frontend version: {version}")

    if already_installed(version):
        log(f"Already installed at {version}, skipping fetch.")
        return

    asset_url = find_tarball_asset(version, config)
    pat = required(config, "GITHUB_PAT")

    with tempfile.TemporaryDirectory(prefix="<frappe-app>-frontend-") as tmpdir:
        tarball = Path(tmpdir) / "frontend.tar.gz"
        download_asset(asset_url, pat, tarball)
        extract_tarball(tarball, PUBLIC_MOBILE_DIR)

    copy_index_to_www()
    STAMP_FILE.write_text(version)
    log(f"Frontend {version} installed.")

if __name__ == "__main__":
    try:
        main()
    except SystemExit:
        raise
    except Exception as e:
        log(f"ERROR: {e}")
        sys.exit(1)
```

</details>

The script has four behaviors: it is **opt-in** (silent when `.env` is absent or
`FRONTEND_VERSION` is empty, loud on any other misconfiguration); **idempotent** (it
stamps the installed version into `.version`; delete the stamp to force a re-fetch);
it **refuses world-readable secrets** (it bails unless `.env` is mode `0600`); and it
uses the **standard library only**.

Wire up the build script. `bench build` runs `npm run build` for any app whose
`package.json` has a `build` script, so create `<frappe-app>/package.json` at the
**app root** (next to `pyproject.toml`):

```json
{
  "name": "<frappe-app>",
  "private": true,
  "scripts": {
    "build": "python3 <frappe-app>/scripts/fetch_frontend.py"
  }
}
```

**Part 2 is done** when `mobile.py`, the `website_route_rules` entry, the fetch script,
and `package.json` are committed and pushed to `<frappe-app>`. The app now defines the
`/mobile` route and the `.env` contract part 3 fills in (`GITHUB_OWNER`, `GITHUB_REPO`,
`GITHUB_PAT`, `FRONTEND_VERSION`) — but nothing fetches yet; the download happens on the
bench during `bench build`.

## 3. Install and serve on the bench

This part runs on each bench that should serve `/mobile`. You give the bench its
GitHub credentials and version pin, build (which runs part 2's fetch script against
part 1's release), and install the app on each site. Repeat per bench.

### The secrets file

Each bench needs a `.env` holding GitHub credentials and the pinned version; the fetch
script reads it during `bench build`. From the root of whichever bench you're
configuring, create the directory and file with restrictive permissions:

```bash
cd ~/frappe/frappe-bench

mkdir -p .<frappe-app>
chmod 700 .<frappe-app>

cat > .<frappe-app>/.env <<'EOF'
FRONTEND_VERSION=<angular-app>-v1.4.0

GITHUB_OWNER=<github-org>
GITHUB_REPO=<monorepo>
ASSET_NAME_PREFIX=<angular-app>

GITHUB_PAT=github_pat_REPLACE_WITH_YOUR_FINE_GRAINED_PAT
EOF

chmod 600 .<frappe-app>/.env
```

- `FRONTEND_VERSION` — the Angular release tag this bench fetches. Pinning is
  per-bench, so production and staging on one server can sit on different versions.
  Leave it unset and `bench build` skips the fetch silently.
- `GITHUB_OWNER` / `GITHUB_REPO` — your org and the monorepo's name.
- `ASSET_NAME_PREFIX` — the tarball prefix (`<angular-app>` matches
  `<angular-app>-v1.4.0.tar.gz`).
- `GITHUB_PAT` — a token with read access to the monorepo (reusable across benches).
- Any of these can be overridden by an environment variable of the same name.

> ⚠️ The mode **must** be `0600` — the fetch script aborts if any group or other bits
> are set.

Generate the PAT in GitHub under **Settings → Developer settings → Personal access
tokens → Fine-grained tokens → Generate**: repository access scoped to the
`<frappe-app>` and `<monorepo>` repos; **Repository permissions → Contents →
Read-only**; expiration of e.g. 90 days, with a rotation reminder on the calendar.

### Install and build

Pull `<frappe-app>` onto the bench at the released tag, then build:

```bash
cd ~/frappe/frappe-bench
bench get-app --branch main git@github.com:<github-org>/<frappe-app>.git

cd apps/<frappe-app>
git fetch --tags
git checkout v1.4.0

cd ~/frappe/frappe-bench
bench setup requirements
bench build
```

> 💡 First private repo on this server? `bench get-app` clones over SSH — add a
> read-only GitHub **deploy key** for the repo first (see
> [Production Deployment](02-production-deployment.md#optional-add-a-private-app)).

The build is working when the output shows the `[fetch_frontend]` lines through to
`Frontend <angular-app>-v1.4.0 installed.`:

```
 DONE  Total Build Time: 410ms
[fetch_frontend] Pinned frontend version: <angular-app>-v1.4.0
[fetch_frontend] Downloading https://api.github.com/repos/...
[fetch_frontend] Downloaded 4.2 MB to /tmp/<frappe-app>-frontend-.../frontend.tar.gz
[fetch_frontend] Extracting to .../apps/<frappe-app>/<frappe-app>/public/mobile
[fetch_frontend] Copied index.html → .../apps/<frappe-app>/<frappe-app>/www/mobile.html
[fetch_frontend] Frontend <angular-app>-v1.4.0 installed.
```

This installs the assets and the shell into the app:

```
apps/<frappe-app>/<frappe-app>/
├── public/mobile/        ← extracted tarball → served at /assets/<frappe-app>/mobile/*
│   └── .version          ← stamp file (e.g. "<angular-app>-v1.4.0")
└── www/mobile.html       ← copy of index.html → served at /mobile
```

If the build skips the fetch, fails to download, or prints no `[fetch_frontend]`
lines at all, see [Troubleshooting](#5-troubleshooting).

### Install on sites and verify

The app is on the bench; each site that should serve `/mobile` still needs it
installed.

> ⚠️ Set `host_name` **before** `install-app` — installation generates URLs from
> `host_name`, so installing first bakes in stale ones.

Repeat the `set-config` and `install-app` pair for each site, then restart once:

```bash
cd ~/frappe/frappe-bench
bench --site a.example.com set-config host_name "https://a.example.com"
bench --site a.example.com install-app <frappe-app>
bench restart
```

Verify the route. The first URL is the entry point; the second is a deep link that
proves the catch-all rule works. Each should answer `HTTP/2 200` with
`content-type: text/html`:

```bash
curl -I https://a.example.com/mobile
curl -I https://a.example.com/mobile/sign-in
```

Then open `https://a.example.com/mobile/sign-in` and confirm the SPA renders. If the
deep link returns 404, see [Troubleshooting](#5-troubleshooting).

With the route serving, confirm the PWA assets resolve live. The manifest and icons
should resolve under the assets root, `/mobile` should return HTML, and `/mobile/`
should 301-redirect (that last redirect is why the manifest scope has no trailing
slash):

```bash
curl -s  https://<host>/assets/<frappe-app>/mobile/manifest.prod.webmanifest | head -10
curl -sI https://<host>/assets/<frappe-app>/mobile/assets/icon/icon-192.png | head -3
curl -sI https://<host>/mobile | head -3
curl -sI https://<host>/mobile/ | grep -iE "^(HTTP|location)"
```

To inspect on a real device, connect over USB with debugging on, visit
`chrome://inspect/#devices` from the laptop, open the PWA → **inspect** →
**Application → Manifest**, and read the **Installability** panel for blockers.

To disable on one tenant, run
`bench --site b.example.com uninstall-app <frappe-app>` — its `/mobile` then returns
404, and other sites are unaffected. Re-enable it by running `install-app` again
(with `host_name` set first) and `bench restart`.

**Part 3 is done** when `https://a.example.com/mobile/sign-in` renders the SPA and the
PWA checks above resolve — the goal this guide set out to reach.

## 4. Routine operations

### Release a frontend version

This is frontend-only — no Frappe-app commit, no migration. Tag and push in the
monorepo (CI builds the tarball), then bump the bench's pin:

```bash
git tag <angular-app>-v1.5.0
git push origin <angular-app>-v1.5.0

cd ~/frappe/frappe-bench
sed -i 's|^FRONTEND_VERSION=.*|FRONTEND_VERSION=<angular-app>-v1.5.0|' .<frappe-app>/.env
bench build
bench restart
```

The release is live when `curl -sI https://a.example.com/mobile | head -3` succeeds
and the loaded JS bundle hashes match the new build. The pin is per-bench, so this
bumps only the bench you run it on.

### Roll back the frontend

Rolling back is a pin change, since each release is an immutable tarball still on
GitHub. The `.version` stamp no longer matches, so the script re-downloads the older
tarball:

```bash
cd ~/frappe/frappe-bench
sed -i 's|^FRONTEND_VERSION=.*|FRONTEND_VERSION=<angular-app>-v1.4.0|' .<frappe-app>/.env
bench build
bench restart
```

To force a re-fetch of the same version (corrupted extraction, or a wiped
`public/mobile/`), delete the stamp:

```bash
rm ~/frappe/frappe-bench/apps/<frappe-app>/<frappe-app>/public/mobile/.version
cd ~/frappe/frappe-bench
bench build
```

### Rotate the PAT

Rotate before expiry, on a suspected leak, or when someone with access leaves. Mint a
fresh fine-grained PAT with the same scopes (`<frappe-app>` + `<monorepo>`, Contents:
Read-only), then replace `GITHUB_PAT=` in `.<frappe-app>/.env` on every bench. There
is nothing to build or restart — each `bench build` reads the PAT as it starts, and
the `.version` stamp still matches so no new download happens.

### Add a second app from the same monorepo

To add a second app (say `admin-app`), add a release workflow keyed to `admin-app-v*`.
On the Frappe side, either run one Frappe app per Angular app (cleanest separation,
most boilerplate) or generalize the fetch script to take a list of apps, each with its
own pin and prefix. Configure it via a per-app secrets file
(`<bench>/.admin-app/.env`) or distinct variable names in the shared `.env`. One PAT
can serve both — add the second repo to its access list.

## 5. Troubleshooting

Most failures surface during `bench build` (the fetch script) or on the first request
to `/mobile`.

- **`Mobile frontend not configured … Skipping`** — `.<frappe-app>/.env` doesn't
  exist. Create it ([the secrets file](#the-secrets-file)).
- **`Mobile frontend disabled … Skipping`** — `FRONTEND_VERSION` is empty or missing.
  Set it in `.env`.
- **`mode 0o644; expected 0o600`** — the secrets file is world-readable. Run
  `chmod 600 .<frappe-app>/.env`.
- **`GitHub API 401`** — the PAT expired or has the wrong scopes. Regenerate it with
  Contents: Read.
- **`GitHub API 404`** — `FRONTEND_VERSION` doesn't match a Release tag. Confirm the
  tag exists in GitHub Releases.
- **No `[fetch_frontend]` lines at all** — `package.json` is missing or `build` is
  misspelled. Check `apps/<frappe-app>/package.json`, then run the script directly:
  `python3 apps/<frappe-app>/<frappe-app>/scripts/fetch_frontend.py`.
- **`/mobile` works but `/mobile/sign-in` 404s** — `website_route_rules` hasn't taken
  effect. Run `bench --site <site> migrate`.

## References

- [Web application manifest](https://www.w3.org/TR/appmanifest/)
- [Service workers](https://www.w3.org/TR/service-workers/)
- [Angular service worker overview](https://angular.dev/ecosystem/service-workers)
