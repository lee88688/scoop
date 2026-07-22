---
name: electron-nsis
description: Use when creating or updating a Scoop manifest for an Electron app built with electron-builder and distributed via GitHub releases. Covers the #/dl.7z NSIS extraction pattern, app-64.7z pre_install, and latest.yml hash autoupdate. Trigger keywords: electron, electron-builder, NSIS installer, latest.yml, GitHub release, app-64.7z, $PLUGINSDIR.
---

# Scoop manifest for electron-builder GitHub releases

Electron apps packaged with electron-builder ship NSIS installers (`.exe`).
Scoop cannot run installers directly, so the standard pattern downloads the
NSIS `.exe`, treats it as a 7z archive via the `#/dl.7z` URL fragment, then
extracts the inner `app-64.7z` (the actual unpacked Electron app) in
`pre_install`. This is the proven pattern used across the ScoopInstaller/Extras
bucket (e.g. motrix, hyper, listen1desktop).

## When to use this skill

- The GitHub release provides `*-win-x64.exe` (and optionally `-win-arm64.exe`)
  produced by electron-builder (look for a `latest.yml` asset in the release).
- There may also be a `*.zip` portable build, but the NSIS + `#/dl.7z` approach
  is preferred because `latest.yml` gives clean, versioned hash autoupdate.
  Only use the `.zip` directly when there is no `latest.yml` / hash source.

## Manifest template (64-bit only)

```json
{
    "version": "<VERSION>",
    "description": "<APP DESCRIPTION>",
    "homepage": "<APP HOMEPAGE>",
    "license": "<SPDX LICENSE>",
    "architecture": {
        "64bit": {
            "url": "https://github.com/<owner>/<repo>/releases/download/v<VERSION>/<Product>-<VERSION>-win-x64.exe#/dl.7z",
            "hash": "sha512:<HEX FROM latest.yml>",
            "pre_install": [
                "Expand-7zipArchive \"$dir\\`$PLUGINSDIR\\app-64.7z\" \"$dir\"",
                "Remove-Item \"$dir\\`$*\", \"$dir\\Uninst*\" -Recurse"
            ]
        }
    },
    "shortcuts": [
        [
            "<Product>.exe",
            "<Product>"
        ]
    ],
    "checkver": {
        "github": "https://github.com/<owner>/<repo>"
    },
    "autoupdate": {
        "architecture": {
            "64bit": {
                "url": "https://github.com/<owner>/<repo>/releases/download/v$version/<Product>-$version-win-x64.exe#/dl.7z",
                "hash": {
                    "url": "$baseurl/latest.yml",
                    "regex": "win-x64\\.exe[\\s\\S]*?sha512:\\s+$base64"
                }
            }
        }
    }
}
```

## Key fields explained

- **`url`**: NSIS installer `.exe` with `#/dl.7z` suffix. The fragment makes
  Scoop save the download as `dl.7z` and extract it with 7-Zip instead of
  running the installer. The download hash still matches the original `.exe`.
- **`pre_install`**: 
  - `Expand-7zipArchive \"$dir\\`$PLUGINSDIR\\app-64.7z\" \"$dir\"` —
    extracts the inner Electron app archive. Note the backtick before
    `` `$PLUGINSDIR ``: `$PLUGINSDIR` is a literal NSIS folder name, the
    backtick escapes the `$` so PowerShell does not treat it as a variable.
  - `Remove-Item \"$dir\\`$*\", \"$dir\\Uninst*\" -Recurse` — cleans up NSIS
    scaffolding (the `$PLUGINSDIR` folder, uninstaller, etc.).
- **`hash`**: `sha512:<hex>`. electron-builder's `latest.yml` stores the
  sha512 in **base64**; Scoop auto-converts base64 → hex and prepends
  `sha512:` via the `$base64` regex template (length 128 → sha512).
- **`bin` vs `shortcuts`**: Electron apps are GUI apps. Prefer `shortcuts`
  (Start Menu entry) only; skip `bin` to avoid console-window shims. Add
  `bin` only if the app ships a real CLI tool alongside the GUI exe.

## Adding arm64

Provide an `arm64` block mirroring `64bit`, using:
- URL: `...-win-arm64.exe#/dl.7z`
- Inner archive: `app-arm64.7z` (verify the name by listing the extracted
  `$PLUGINSDIR` contents — it may be `app-arm64.7z` or `app-64.7z`).
- Hash regex: `win-arm64\\.exe[\\s\\S]*?sha512:\\s+$base64`

```json
"arm64": {
    "url": "https://github.com/<owner>/<repo>/releases/download/v$version/<Product>-$version-win-arm64.exe#/dl.7z",
    "hash": {
        "url": "$baseurl/latest.yml",
        "regex": "win-arm64\\.exe[\\s\\S]*?sha512:\\s+$base64"
    },
    "pre_install": [
        "Expand-7zipArchive \"$dir\\`$PLUGINSDIR\\app-arm64.7z\" \"$dir\"",
        "Remove-Item \"$dir\\`$*\", \"$dir\\Uninst*\" -Recurse"
    ]
}
```

## How to gather the required values

1. **Latest version + assets**: query the GitHub API
   `https://api.github.com/repos/<owner>/<repo>/releases/latest` — gives
   `tag_name` (strip the leading `v` for `version`), asset names, sizes, and
   `digest` fields (`sha256:...`).
2. **`latest.yml`** (electron-builder metadata, in the release assets):
   contains `version`, per-asset `sha512` (base64), and `size`. Fetch it via
   `https://github.com/<owner>/<repo>/releases/download/v<VERSION>/latest.yml`.
   It lists multiple files (universal, x64, arm64) — the autoupdate regex must
   target the specific arch entry, not the first one.
3. **Exe name / productName**: read the repo's `electron-builder` config
   (`electron-builder.config.cjs` / `electron-builder.yml` / `package.json`
   `build` block). The `productName` becomes both the install dir exe name
   (`<productName>.exe`) and the release asset prefix.
4. **License / homepage**: from the repo API (`license.spdx_id`,
   `homepage`) or `package.json`.

## autoupdate hash regex — multi-file latest.yml

`latest.yml` lists several files; a naive `sha512:\s+$base64` matches the
**first** entry (often the universal bundle), which is the wrong hash for the
arch-specific installer. Target the correct entry:

```
win-x64\.exe[\s\S]*?sha512:\s+$base64
```

- `[\s\S]*?` is non-greedy, so it matches the **first** `sha512:` after the
  `win-x64.exe` URL line.
- `$base64` is a Scoop regex macro expanding to
  `([a-zA-Z0-9+\/=]{24,88})` — a capturing group; Scoop uses capture group 1
  as the hash value.

## Generating the initial hash

The manifest needs a valid `hash` for the current version. Two options:

1. **Compute from latest.yml** (fast, no large download): take the base64
   sha512 for the x64 entry and convert to hex:
   ```powershell
   $b64 = "<base64 from latest.yml>"
   $hex = ([System.Convert]::FromBase64String($b64) | ForEach-Object { $_.ToString('x2') }) -join ''
   "sha512:$hex"
   ```
2. **Let checkver fill it**: set a lower `version`, run
   `.\bin\checkver.ps1 <app> -f` — it downloads `latest.yml`, applies the
   autoupdate regex, and writes the hash into the manifest.

## Verification workflow

```powershell
.\bin\formatjson.ps1                      # normalize JSON formatting
$env:HTTP_PROXY="http://127.0.0.1:10808"  # if GitHub direct downloads are blocked
$env:HTTPS_PROXY="http://127.0.0.1:10808"
.\bin\checkver.ps1 <app>                  # version detection (prints: <app>: <version>)
# verify autoupdate end-to-end:
#   temporarily lower "version" in the manifest, then:
.\bin\checkver.ps1 <app> -f              # should bump version + write correct hash
# real install test (user-run, downloads ~100-270 MB):
scoop install .\bucket\<app>.json
```

## Reference example

`bucket/netcatty.json` in this repo is a complete working example of this
pattern (Netcatty, an Electron SSH client). The autoupdate hash regex
`win-x64\.exe[\s\S]*?sha512:\s+$base64` was verified to fetch and convert the
correct base64 sha512 → hex from the multi-file `latest.yml`.
