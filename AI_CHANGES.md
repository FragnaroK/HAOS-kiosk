# AI-Generated Changes Log

All changes in this file were generated with GitHub Copilot (Claude Sonnet 4.6) during an
interactive session on 2026-04-18. Changes target the `testing-chromium` branch and address
WebRTC camera-grid support, MSE flicker, and general browser stability for Chromium kiosk mode.

---

## Summary of Changes

### Problem Addressed

The add-on used Luakit (webkit2gtk) as its sole browser. Luakit does not expose WebRTC or
MSE APIs in a way that allows Home Assistant camera cards using RTC streams to work reliably.
MSE streams flickered due to missing compositor, page refresh interruptions, and no GPU hints.

### Solution

Switch default browser to Chromium kiosk mode with:
- HA-origin-scoped WebRTC permission policy.
- Managed policy suppression of common kiosk-breaking popups.
- Auto-login keystroke fallback for Chromium (since Luakit's Lua auto-login does not apply).
- Optional compositor autodetection for MSE flicker reduction.
- Luakit retained as a selectable fallback mode.

---

## File Changes

### `haoskiosk/Dockerfile`

- Added `chromium` to the `apk add` package list as the new default browser.
- Removed `xcompmgr` from the package list — not available in the HAOS Alpine base image (build fix).

### `haoskiosk/config.yaml`

New options added:

| Option | Default | Description |
|---|---|---|
| `browser_mode` | `chromium` | Select `chromium` (default) or `luakit` (legacy). |
| `chromium_flags_extra` | `""` | Extra Chromium CLI flags for advanced tuning. |
| `chromium_auto_login` | `true` | Auto-fill/submit HA login form via xdotool after startup. |
| `chromium_login_retries` | `2` | Number of login attempts (increase on slow systems). |
| `webrtc_autogrant_ha_only` | `true` | Auto-grant camera/mic only for HA origin URL. |
| `enable_compositor` | `false` | Enable compositor to reduce screen tearing. |
| `compositor_cmd` | `""` | Custom compositor command; auto-detected if empty. |
| `mse_profile` | `balanced` | Chromium rendering profile: `compat`, `balanced`, or `smooth`. |

Existing `command_whitelist` updated to include `chromium` and `chromium-browser`.

### `haoskiosk/run.sh`

#### Browser mode selection (new)
- `BROWSER_MODE` config variable controls whether Chromium or Luakit is launched.
- Chromium binary is resolved at runtime (`chromium-browser` → `chromium` fallback).
- HA URL origin (`HA_ORIGIN`) is derived from `HA_URL` for scoped policy and permission use.

#### Chromium kiosk flags (new)
Default Chromium flags set via `CHROMIUM_FLAGS_DEFAULT`:
- `--kiosk` — Full-screen kiosk mode.
- `--no-first-run`, `--no-default-browser-check` — Suppress first-run dialogs.
- `--disable-extensions` — No extension UI.
- `--disable-features=Translate,OptimizationHints,ChromeWhatsNewUI,DiscoverFeed` — Suppress UI nudges and the "Shortcuts" new-tab popup.
- `--disable-save-password-bubble` — Suppresses the "Save password?" prompt.
- `--autoplay-policy=no-user-gesture-required` — Allows stream autoplay.
- `--disable-background-timer-throttling`, `--disable-backgrounding-occluded-windows`, `--disable-renderer-backgrounding` — Prevents stream throttling.
- `--enable-gpu-rasterization`, `--ignore-gpu-blocklist`, `--enable-zero-copy` — GPU rendering hints.
- `--user-data-dir=/tmp/chromium-kiosk` — Isolated, ephemeral profile.

Per-profile additions via `MSE_PROFILE`:
- `compat`: adds `--disable-gpu` for unstable GPU stacks.
- `smooth`: adds `--enable-features=VaapiVideoDecoder,CanvasOopRasterization`.
- `balanced` (default): adds `--enable-features=VaapiVideoDecoder`.

#### Chromium managed policy (new/updated)
A managed policy JSON is written to `/etc/chromium/policies/managed/haoskiosk-webrtc.json`
at startup for all Chromium builds. It controls:
- `AudioCaptureAllowed` / `VideoCaptureAllowed` — Governed by `webrtc_autogrant_ha_only`.
- `AudioCaptureAllowedUrls` / `VideoCaptureAllowedUrls` — Scoped to `HA_ORIGIN`.
- `AutoplayAllowed: true` — Camera streams autoplay without user gesture.
- `PasswordManagerEnabled: false` — Disables "Save password?" popup.
- `AutofillAddressEnabled: false`, `AutofillCreditCardEnabled: false` — Suppresses autofill popups.
- `PromotionalTabsEnabled: false` — No promotional/tip tabs.
- `BrowserSignin: 0` — No sign-in prompts.
- `DefaultNotificationsSetting: 2` — Block notification requests.
- `DefaultPopupsSetting: 2` — Block popup windows.
- `BookmarkBarEnabled: false` — Clean kiosk UI.
- `NewTabPageLocation` — Points new tabs to HA origin.

#### Chromium auto-login (new)
When `chromium_auto_login=true`:
- Waits for Chromium window to appear after browser launch.
- Uses `xdotool` to fill username, Tab to password, and Return to submit.
- Retries up to `chromium_login_retries` times, each separated by `login_delay` seconds.
- Runs as a non-blocking background job so startup is not delayed.

#### Optional compositor (new)
When `enable_compositor=true`:
- If `compositor_cmd` is empty, auto-detects in order: `picom`, `compton`, `xcompmgr`.
- Resolves binary from autodetected or explicit command before launching.
- Logs warning and continues cleanly if no compositor binary is found.

#### Refresh warning (new)
When `browser_mode=chromium` and `browser_refresh > 0`, a startup warning is logged
recommending `browser_refresh=0` for camera-heavy dashboards.

#### URL join fix (new)
`TARGET_URL` is now built with `${HA_URL%/}/${HA_DASHBOARD#/}` to prevent double-slash
when `HA_DASHBOARD` starts with `/`.

#### Browser process monitor fix (new)
`pgrep` pattern changed from `^$BROWSER` (path-anchored) to `$BROWSER_NAME` (basename)
so Chromium subprocesses are correctly detected and the watchdog loop exits cleanly.

#### Cleanup (updated)
`rm -rf /tmp/chromium-kiosk` added to cleanup handler to remove ephemeral Chromium profile on exit.

### `haoskiosk/xorg.conf.default`

Added to the `Device` section for the modesetting driver:
- `Option "TearFree" "true"` — Reduces display tearing during video playback.
- `Option "TripleBuffer" "true"` — Reduces frame drop on high-refresh content.
- `Option "SwapbuffersWait" "true"` — Synchronises buffer swaps with vblank.

### `haoskiosk/README.md`

Documented all new configuration options:
- Browser Mode, Chromium Flags Extra, Chromium Auto Login, Chromium Login Retries.
- WebRTC Auto-Grant (HA Only), MSE Profile, Enable Compositor, Compositor Command.
- Added note recommending `browser_refresh: 0` for camera-heavy dashboards.

---

## Known Limitations / Follow-Up Notes

- No Intel VAAPI/GPU decode packages are installed. On Intel iGPU hardware the `smooth` MSE
  profile will attempt VAAPI but may fall back to software decode silently.
- Compositor autodetection probes `picom`/`compton`/`xcompmgr` but none are installed in the
  default image. `enable_compositor` only works if the user installs a compositor manually or
  via `chromium_flags_extra` / `xorg_conf` workarounds.
- Auto-login via xdotool is best-effort keystroke injection and may need `login_delay`
  tuning on slower hardware.
