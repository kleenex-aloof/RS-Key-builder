# RS-Key Builder

Build [RS-Key](https://github.com/TheMaxMur/RS-Key) firmware — no local toolchain needed.

## Usage

1. **Create a repo** from this template (green "Use this template" button).
1. **Enable GitHub Actions** in your new repo (Settings → Actions → Allow all actions).
1. **Trigger a build** — go to Actions → "Build RS-Key firmware" → "Run workflow".
   Fill in the build parameters.
1. **Download** the resulting `firmware.uf2` from the created GitHub Release.
1. **Flash** your RP2350 board:
   ```
   picotool load -v firmware.uf2 && picotool reboot
   ```
   Or hold BOOTSEL, plug in, and drag-and-drop the UF2 onto the `RP2350` volume.
1. **Delete the repo** — it served its purpose.

## Build inputs

| Input | Default | Description |
|---|---|---|
| `RS_KEY_VERSION` | `latest` | Git ref to build (tag, branch, commit, or `latest`) |
| `BOARD` | `waveshare-rp2350-one` | Board hardware preset (`custom` for manual) |
| `VIDPID` | `RSKey` | USB identity (RSKey, Yubikey5, NitroFIDO2, etc.) |
| `FW_VERSION` | `5.7.4` | Firmware version string |
| `PRESENCE_PIN` | `bootsel` | Touch button GPIO |
| `FEATURES` | — | Cargo features, comma-separated (e.g. `no-touch,display`) |
| `ENV_EXTRA` | — | `KEY=VAL` pairs for rare knobs |

All env vars from [RS-Key build.md](https://github.com/TheMaxMur/RS-Key/blob/main/docs/build.md) are
available — either as dedicated inputs or through `ENV_EXTRA` for rare knobs. Board presets
pre-fill hardware values; individual inputs override the preset when filled.

## Flashing

Put your RP2350 board into BOOTSEL mode (hold the BOOT button while plugging in USB):

```
picotool load -v firmware.uf2 && picotool reboot
```

Or copy `firmware.uf2` to the mounted `RP2350` volume.

Quick start guide: https://github.com/TheMaxMur/RS-Key/blob/main/docs/quickstart.md

## Secure boot (optional)

If you add a signing key as a `FIRMWARE_SIGNING_KEY` repository secret, the workflow will
produce two additional assets in the Release:

- `firmware-sealed.uf2` — firmware signed with your key.
- `otp_secureboot.json` — OTP boot-key fingerprint for fuse provisioning.

### Generate a signing key

The key must be a **secp256k1 ECDSA private key** in PEM format. Generate one with:

```
openssl ecparam -genkey -name secp256k1 -out firmware-key.pem
```

You will get something like this:

```
-----BEGIN EC PARAMETERS-----
BgUrgQQACg==
-----END EC PARAMETERS-----
-----BEGIN EC PRIVATE KEY-----
MHQCAQEEIIlVz0CBAqFi67SRLLkDpYkGnagKBAa1U/PrcLQQxf8yoAcGBSuBBAAK
oUQDQgAE4bkPcXvOx71UZgRV7NA14+IIIRW95Yxwx6BoG//qPtkG26Yfk9TITX4V
OZ+xoIjoPLrDT/RinJHvJoZPnHao7Q==
-----END EC PRIVATE KEY-----
```

> [!IMPORTANT]
> This key is the root of trust for your device. Once fused into OTP, a different key
> cannot be used. Keep it secret, back it up, and never commit it.

### Add the secret

1. In your repo: **Settings → Secrets and variables → Actions → New repository secret**.
1. Name: `FIRMWARE_SIGNING_KEY`
1. Paste the full contents of `firmware-key.pem` (including `-----BEGIN EC PRIVATE KEY-----`
   and `-----END EC PRIVATE KEY-----` lines).
1. Save.

### Enable secure boot on the board

Flash `firmware-sealed.uf2` via BOOTSEL as usual, then follow the upstream production guide
which covers OTP provisioning and verification:

https://github.com/TheMaxMur/RS-Key/blob/main/docs/production.md

Secure boot is **irreversible** once OTP fuses are set.
