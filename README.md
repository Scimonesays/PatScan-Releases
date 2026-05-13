# PatScan-Releases — sideload updater channel

This **public** repository holds the small **HTTPS manifest** and **APK files** PatScan uses for **in-app sideload updates**. It is the companion to the private **PatScan** source repository: keep secrets and app source out of here; publish only what users need to download and what the updater reads.

## What lives on `main`

| Artifact | Role |
|----------|------|
| **`app-update.json`** | Fetched by the app over HTTPS. Describes the latest `versionCode`, human-readable `versionName`, and where to download the APK. |
| **`PatScan-vX.Y.Z.apk`** | The installable package. Filename should match whatever you put in `apkUrl` (recommended: one file per shipped version). |

Stable URLs (replace `main` with a tag or commit SHA only if you intentionally pin):

- **Manifest (raw JSON):**  
  `https://raw.githubusercontent.com/Scimonesays/PatScan-Releases/main/app-update.json`
- **APK (GitHub `raw` redirect):**  
  `https://github.com/Scimonesays/PatScan-Releases/raw/main/PatScan-vX.Y.Z.apk`  
  (equivalent to the `raw.githubusercontent.com/.../PatScan-vX.Y.Z.apk` URL)

Official PatScan builds default to the manifest URL above unless overridden at build time (see the private repo README / `local.properties`).

## `app-update.json` schema

All string URLs **must** use **`https://`**.

```json
{
  "versionCode": 6,
  "versionName": "0.4.2",
  "apkUrl": "https://github.com/Scimonesays/PatScan-Releases/raw/main/PatScan-v0.4.2.apk",
  "releasePageUrl": "https://github.com/Scimonesays/PatScan-Releases"
}
```

| Field | Required | Notes |
|-------|----------|--------|
| **`versionCode`** | Yes | Integer. Must be **greater** than the build installed on the device for the updater to offer “Download & install”. |
| **`versionName`** | Yes | Shown in the UI (e.g. `0.4.2`). Keep it in sync with marketing / tags. |
| **`apkUrl`** | Yes | Direct HTTPS link to the APK file on this repo (or another trusted host). |
| **`releasePageUrl`** | No | Optional; powers “Open downloads page” in the app (e.g. this repo’s home page). |

After you push, GitHub’s CDN may cache the previous JSON briefly. Current PatScan builds reduce stale reads by **cache-busting manifest requests**; if testers still see an old version, wait a minute and tap **Check for update** again or try another network.

## Publisher workflow (each release)

1. **In the PatScan source repo**  
   - Bump **`versionCode`** (monotonic; never reuse a lower value for a “new” release).  
   - Bump **`versionName`** as you prefer (e.g. `0.4.3`).  
   - Align **`release_public/app-update.json`** (or your internal template) with the new **`apkUrl`** filename you plan to ship.

2. **Build an APK**  
   - Example (debug for internal sideload):  
     `app\build\outputs\apk\debug\app-debug.apk`  
   - Or your signed **release** APK from your usual pipeline.

3. **In this repo (`PatScan-Releases`)**  
   - Copy/rename the APK to the filename referenced by **`apkUrl`** (e.g. `PatScan-v0.4.3.apk`).  
   - Update **`app-update.json`** on **`main`** so **`versionCode` / `versionName` / `apkUrl`** match that APK.  
   - Commit and push to **`main`**.  
   - Do **not** delete older APKs unless you intentionally want to break old links; keeping them avoids broken bookmarks.

4. **Verify HTTPS** (expect **200**, not **404**):

   ```powershell
   curl.exe -sI "https://raw.githubusercontent.com/Scimonesays/PatScan-Releases/main/app-update.json"
   curl.exe -sI "https://raw.githubusercontent.com/Scimonesays/PatScan-Releases/main/PatScan-v0.4.3.apk"
   ```

   Adjust the APK path to match `apkUrl`.

## Quick copy script (Windows / PowerShell)

Point `PATSCAN_ROOT` at your PatScan clone. Adjust paths and the destination filename to match `apkUrl`.

```powershell
$PATSCAN_ROOT = "$env:USERPROFILE\Documents\CODE\Patscan"
$RELEASES     = "$env:USERPROFILE\Documents\CODE\PatScan-Releases"

Copy-Item -Force "$PATSCAN_ROOT\app\build\outputs\apk\debug\app-debug.apk" `
  "$RELEASES\PatScan-v0.4.3.apk"
Copy-Item -Force "$PATSCAN_ROOT\release_public\app-update.json" `
  "$RELEASES\app-update.json"

cd $RELEASES
git add app-update.json PatScan-v0.4.3.apk
git commit -m "Release PatScan 0.4.3 (versionCode 7)."
git push origin main
```

Use your real **`versionCode`**, **`versionName`**, and filename in the commit message and paths.

## On-device test

1. Install any build that includes the sideload updater (see private repo: **Settings → App updates**).  
2. **Settings → App updates → Check for update**  
   - If the manifest **`versionCode`** is higher than the installed app, you should see an update and **Download & install**.  
3. If the system blocks install, enable **Install unknown apps** (or equivalent) for PatScan, then try again.

The app **does not** auto-install; it only downloads and opens the system installer when the user asks.

## GitHub limits and hygiene

- Keep APKs under GitHub’s **file size** limits (large releases may need **Git LFS** or an external HTTPS host; if you move the APK off GitHub, keep **`apkUrl`** on **`https://`**).  
- Never commit signing keys, tokens, or private URLs in this repo.

## Forks

If you fork this layout, change **`apkUrl`**, **`releasePageUrl`**, and the PatScan app’s **`UPDATE_MANIFEST_URL`** / **`UPDATE_RELEASE_PAGE_URL`** (via `local.properties` or your fork’s Gradle defaults) to your own HTTPS endpoints.
