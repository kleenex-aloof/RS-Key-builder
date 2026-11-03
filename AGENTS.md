# RS-Key Builder — AGENTS.md

## What this is

A **GitHub template repository** that builds [RS-Key](https://github.com/TheMaxMur/RS-Key) firmware
via GitHub Actions — no local toolchain needed. Users create a repo from this template, configure
build parameters through `workflow_dispatch` inputs, trigger a build, download the resulting
`firmware.uf2` from a GitHub Release, then delete the repo.

Upstream firmware project: https://github.com/TheMaxMur/RS-Key

## Key facts

- **`workflow_dispatch` only** — no push/tag triggers. Repo is ephemeral (one build, then delete).
- **Build step** checks out TheMaxMur/RS-Key source directly (`actions/checkout` with
  `repository: TheMaxMur/RS-Key`). This repo contains zero firmware source — only the workflow.
- **Nix with flakes** (`DeterminateSystems/nix-installer-action`) for the toolchain.
- **Build command** (inside `nix develop`):
  ```
  env VIDPID=... FW_VERSION=... cargo build --release -p firmware [--features ...]
  ```
- **UF2 conversion**: `picotool uf2 convert target/thumbv8m.main-none-eabihf/release/firmware -t elf firmware.uf2`
- **Artifact** uploaded both as a workflow artifact and a GitHub Release asset.

## Build inputs (`workflow_dispatch`)

All knobs map 1:1 to RS-Key's build env vars (see [build.md](https://github.com/TheMaxMur/RS-Key/blob/main/docs/build.md)):

| # | Input | Type | Default | Description |
|---|---|---|---|---|
| 1 | `RS_KEY_VERSION` | string | `latest` | Git ref to build: tag (`v0.3.9`), branch, commit, or `latest` |
| 2 | `BOARD` | choice | `waveshare-rp2350-one` | Board hardware preset (`custom` = manual) |
| 3 | `VIDPID` | choice | `RSKey` | USB identity preset |
| 4 | `FW_VERSION` | string | `5.7.4` | Firmware version string |
| 5 | `PRESENCE_PIN` | string | `bootsel` | Touch button GPIO |
| 6 | `PRESENCE_ACTIVE_HIGH` | choice | (empty) | Presence button polarity |
| 7 | `FLASH_SIZE` | string | (empty) | Board flash size (board preset if empty) |
| 8 | `KVMAIN` | string | (empty) | KV partition size |
| 9 | `LED_KIND` | choice | (empty) | LED driver backend |
| 10 | `LED_PIN` | string | (empty) | Status LED GPIO |
| 11 | `LED_POWER_PIN` | string | (empty) | LED power gate GPIO |
| 12 | `LED_ORDER` | choice | (empty) | WS2812 wire byte order |
| 13 | `USR_LED_PIN` | string | (empty) | Nuisance user LED GPIO |
| 14 | `USR_LED_ACTIVE_HIGH` | choice | (empty) | User LED polarity |
| 15 | `MAX_LEDS` | string | (empty) | Addressable LED chain length |
| 16 | `XOSC_DELAY_MULT` | string | (empty) | Crystal oscillator startup delay |
| 17 | `USB_VID` | string | (empty) | Raw USB VID override |
| 18 | `USB_PID` | string | (empty) | Raw USB PID override |
| 19 | `USB_MANUFACTURER` | string | (empty) | USB descriptor manufacturer |
| 20 | `ENV_EXTRA` | string | (empty) | `KEY=VAL` pairs for rare knobs |
| 21 | `FEATURES` | string | (empty) | Cargo features, comma-separated |

### `ENV_EXTRA` — rare knobs

Space-separated `KEY=VALUE` pairs for env vars without dedicated fields:
`USB_PRODUCT`, `WAKE_PIN`, `WAKE_ACTIVE_HIGH`, `FAKE_MKEK`, `FAKE_DEVK`.

Example: `USB_PRODUCT="My Key" WAKE_PIN=25 WAKE_ACTIVE_HIGH=1`

### Board presets

When `BOARD != custom`, hardware knobs are pre-filled. Individual inputs override.

| Board | FLASH_SIZE | KVMAIN | LED | Other |
|---|---|---|---|---|
| `waveshare-rp2350-one` | 4M | — | ws2812, GPIO16, rgb | — |
| `seeed-xiao-rp2350` | 2M | 896K | ws2812, GPIO22, grb, power=23 | USR_LED=25 |
| `pimoroni-tiny-2350` | 8M | — | pimoroni | — |
| `waveshare-rp2350-zero-cm` | 2M | 896K | ws2812, GPIO16?, rgb? | — |
| `tenstar-rp2350-usb` | 16M | — | ws2812, GPIO22, grb | — |
| `waveshare-rp2350-touch-lcd-2-8` | 16M | — | none | display feature |
| `custom` | (manual) | (manual) | (manual) | — |

## Constraints

- **Do NOT add push/tag triggers** — repo is single-use by design.
- **Do NOT embed RS-Key as a submodule** — checkout upstream directly each build.
- **Do NOT add local dev setup instructions** — this project exists to avoid local tooling.
- **Release is the delivery channel** — the UF2 must end up in a GitHub Release so the user
  can download it before deleting the repo.

## Secure boot sealing

When the `FIRMWARE_SIGNING_KEY` secret exists, the workflow:

1. Writes it to `/tmp/secure_boot_key.pem`
2. Runs `picotool seal --sign --hash firmware.uf2 firmware-sealed.uf2 /tmp/secure_boot_key.pem otp_secureboot.json --major 1 --minor 0`
3. Uploads `firmware-sealed.uf2` + `otp_secureboot.json` as both workflow artifacts and Release assets
4. Appends secure-boot OTP flashing instructions to the Release notes

The signing key is a secp256k1 ECDSA private key in PEM format. Key is never exposed in logs.
Zero new `workflow_dispatch` inputs are added for sealing — `--major 1 --minor 0` is hardcoded,
`--rollback` omitted (available via manual workflow edit if needed).

## Upstream docs reference

- Full build knobs: https://github.com/TheMaxMur/RS-Key/blob/main/docs/build.md
- Quick start / flashing: https://github.com/TheMaxMur/RS-Key/blob/main/docs/quickstart.md
- Hardware guide: https://github.com/TheMaxMur/RS-Key/blob/main/docs/hardware.md
