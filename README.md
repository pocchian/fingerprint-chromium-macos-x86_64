# fingerprint-chromium for Intel macOS (x86_64)

An **x86_64** macOS build of Chromium 152.0.7977.82 with fingerprint spoofing, plus the patches it was built from.

It is [ungoogled-chromium](https://github.com/ungoogled-software/ungoogled-chromium) 152.0.7977.82 with the fingerprint patches from
[adryfish/fingerprint-chromium](https://github.com/adryfish/fingerprint-chromium) ported to 152, and a round of fixes on top (below).

Upstream publishes arm64 builds, so this exists for Intel Macs and Intel macOS VMs.

## Download

The dmg is on the [Releases](../../releases) page. It is **ad-hoc signed and not notarized**, so macOS will not open it by default.

```sh
# the dmg contains Chromium.app; copy it to /Applications, renaming it if you
# already have a Chromium there, then clear the quarantine flag on that copy:
xattr -cr "/Applications/Chromium.app"
```

Verify what you downloaded:

```sh
shasum -a 256 ungoogled-chromium_152.0.7977.82-1.1_x86_64-macos-adhoc-tellsfix.dmg
# 95177259f4f86ef09c5a8690230fce5c3de2c8f140f3128c94df06383ebf55e7
```

If you would rather not trust someone else's binary, the patches here are everything you need to build your own with
[ungoogled-chromium-macos](https://github.com/ungoogled-software/ungoogled-chromium-macos): drop `patches/extra/fingerprint/` into the wrapper's patch set and build with `target_cpu="x64"`.

## Running it

```sh
open -n -a "/Applications/Chromium 152.app" --args \
  --user-data-dir="$HOME/fp-profiles/one" \
  --fingerprint=2101 --fingerprint-platform=macos --fingerprint-brand=Chrome \
  --fingerprint-gpu-model="A18 Pro" --fingerprint-hardware-concurrency=6 \
  --fingerprint-platform-version=26.6.2 --fingerprint-device-memory=8
```

- **One seed per identity.** The same `--fingerprint` seed always produces the same canvas, WebGL and audio fingerprints, so keep a seed and its `--user-data-dir` together. Two identities that share a seed are linkable.
- **On a machine with no GPU** (a VM), add `--use-angle=swiftshader --enable-unsafe-swiftshader` for WebGL, and `--use-mock-keychain` if page loads hang waiting on the keychain.
- `--disable-spoofing=canvas,audio,font,gpu,clientrects` turns individual pieces off.

## What this adds over the upstream patches

Fixed here after testing each behaviour against an unmodified build:

- **`--fingerprint-device-memory`**: `navigator.deviceMemory`, workers and both `Device-Memory` request headers now report one value. Previously JS said 8 while the headers leaked the machine's real RAM.
- **`measureText`**: the noise factor was applied as a multiplier, collapsing every text metric to ~0 (a 79 px string measured 0.00003 px). Removed.
- **Canvas/WebGL pixel noise, rewritten**: an unparenthesised address macro made the "edge pixel" test read misaligned bytes, so even a solid-colour canvas came back modified. It is now idempotent, leaves uniform images byte-identical, keys noise to absolute canvas coordinates, and covers `getImageData`, `toDataURL`, `toBlob`, `convertToBlob` and `readPixels` consistently.
- **Encoded output**: `toBlob` and `OffscreenCanvas.convertToBlob` were not noised at all, and the noised copy dropped the colour space, so JPEG and WebP lost their ICC profile. Both fixed; encoded bytes now match an unmodified build for unnoised content.
- **Element rectangles and fonts**: the ±0.001 px `getBoundingClientRect` offset and the "hide 2% of fonts" rule are gone. Both were detectable in one line of JavaScript (real layout values are multiples of 1/64; real Macs have all their system fonts) and bought nothing.
- **Audio**: constant sources, silence and square-wave plateaus stay exact; the noise only touches varying samples, at about one part in a million.
- **Two upstream bugs**: a red/blue channel swap on BGRA buffers, and a `readPixels` path that walked 4 bytes per pixel through 1–2 byte buffers.

## State

- [Pixelscan](https://pixelscan.net) reports **consistent, no masking detected** for five different Apple silicon Mac profiles. Before these fixes it flagged all five.
- Two independently written verifiers each pass 29 of 29 checks covering user agent and client hints, canvas, WebGL, audio, fonts, element geometry, device memory, version consistency and per-profile stability.
- Known gaps: screen size is **not** spoofed (the switches exist upstream but nothing reads them); WebGL limits are whatever the host renderer reports; the Windows profile's font set is inconsistent; canvas and audio noise depend only on the seed, not the spoofed platform.

## Credits and licence

- [ungoogled-chromium](https://github.com/ungoogled-software/ungoogled-chromium) and [ungoogled-chromium-macos](https://github.com/ungoogled-software/ungoogled-chromium-macos) — BSD-3-Clause
- [adryfish/fingerprint-chromium](https://github.com/adryfish/fingerprint-chromium) — BSD-3-Clause, the origin of the fingerprint patches ported here
- Chromium itself — BSD-3-Clause and the licences listed at `chrome://credits` in the browser

The patches in this repository are BSD-3-Clause; see `LICENSE`. Provided as is, with no warranty.
