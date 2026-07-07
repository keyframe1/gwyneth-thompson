# Media Processing Notes — Gwyneth / Nonnie & Team

How the website media assets in `public/media/` were produced from the raw phone
footage, so **future footage can be graded and cut to match** this exact look.
Everything was done with `ffmpeg` / `ffprobe` (ffmpeg 8.1).

---

## 1. Source footage

- **Camera:** Samsung S25 Ultra, 10 clips (`20260629_*.mp4`) + studio headshots (`IMG_37xx.jpg`).
- **Video specs (all clips):** HEVC/H.265, **3840×2160 @ 30 fps**, `yuv420p`.
- **Color:** `bt709` primaries/transfer — i.e. **standard SDR, NOT HDR/HLG**, so
  **no tone-mapping is needed**. (If a future clip probes as `arib-std-b67` / `smpte2084`,
  add a tonemap stage — see §7.)
- **Orientation gotcha:** every clip carries a **`rotation=-90`** display matrix, so the
  *displayed* frame is **portrait 2160×3840** (shot holding the phone upright). ffmpeg
  auto-rotates before filters, so all crops below operate on a **2160×3840** frame.
- Raw originals are **git-ignored** (see `.gitignore`). Only optimized assets are committed.

Inventory quick-ref (content): the plaid-blazer "showing a home" set — `162648`, `161949`,
`162617`, `162109`, `161845` — is the cohesive wardrobe used for the hero. `1637xx`/`163941`
(white blouse + taupe corset, on phone) is a *different outfit/day* → reserved for social only.
`161055` / `161305` are empty-property B-roll (no subject).

---

## 2. The grade (identical on every clip — this is the "look")

S25 footage is oversharpened + slightly oversaturated. The grade calms it toward a
**warm, filmic, editorial** look matching the earthy-botanical brand
(earth `#3B2F22`, clay `#9E6B4A`, sage `#5A6B4F`, sand `#E8DFD1`). It is deliberately
**subtle** — if it reads as "graded," it's too much.

**Exact filter string (copy verbatim for new footage):**

```
eq=saturation=0.91:contrast=1.035,
curves=master='0/0.015 0.25/0.238 0.5/0.5 0.75/0.77 1/0.968',
colorbalance=rs=0.012:rm=0.032:rh=0.005:gm=0.006:bs=-0.010:bm=-0.032:bh=-0.010,
unsharp=5:5:-0.26
```

What each part does:
- `eq=saturation=0.91:contrast=1.035` — gentle desaturation + a touch of contrast.
- `curves` — lifts blacks to ~0.015 (soft matte), mild S-curve in the mids, rolls highlights
  to ~0.968 (filmic, not clipped).
- `colorbalance` — warms midtones (red up, blue down), keeps highlights mostly clean so
  whites read warm-cream, not yellow; a hair of warmth in the shadows.
- `unsharp=5:5:-0.26` — **negative** amount = softens the digital over-sharpening.

No teal-and-orange. Grain was **not** baked in (it fights the size budget); instead a light
`gradfun` de-band is applied at encode time (§4) to keep the big flat white walls clean.

---

## 3. Framing (vertical source → landscape 16:9 hero)

Each hero beat is a **native 1920×1080 crop** taken straight out of the rotated 2160×3840
frame — **no scaling**, so it stays razor sharp and both axes are recomposable:

```
crop=1920:1080:X:Y      # X ∈ [0,240] (h-recenter), Y ∈ [0,2760] (v-position)
```

Tune **Y so her eyeline lands in the upper third** (~35% down). Full-body shots needed
`Y ≈ 1000`; tighter shots `Y ≈ 470–600`. Verify by extracting a mid-slice frame before
committing — don't eyeball the number.

**Exception — wide gestures** (e.g. the arms-open "welcome"): a 1920 crop clips the spread
hands, so that one beat uses full width then scales:
`crop=2160:1215:0:960,scale=1920:1080:flags=lanczos`.

Per-clip intermediates were encoded `libx264 -crf 15` (visually lossless) before assembly.

---

## 4. The hero supercut

7 beats, ~3 s each, **0.35 s cross-dissolves** (`xfade=transition=fade`), **silent**,
opens & closes at the bright-windows location for a clean loop. **Final length ≈ 19.1 s.**

| # | Beat | Source clip | in → dur | crop |
|---|------|-------------|----------|------|
| 1 | Confident stand + smile (windows) — hook | `162648` | 8.4s +3.4 | X110 Y1010 |
| 2 | Walking through the home — motion | `162109` | 16.8s +2.8 | X120 Y1020 |
| 3 | Hands-up "presenting" — energy | `162617` | 2.4s +2.6 | X120 Y470 |
| 4 | Warm smile at the range — kitchen | `162109` | 42.0s +3.4 | X40 Y1000 |
| 5 | Frontal talk — connection | `161845` | 8.0s +3.0 | X90 Y600 |
| 6 | Arms-open "welcome" — climax | `161949` | 1.0s +3.4 | full-width Y960 |
| 7 | Gesturing at window — loop-back close | `162648` | 0.2s +2.6 | X120 Y560 |

xfade offsets (d=0.35): 3.05, 5.50, 7.75, 10.80, 13.45, 16.50.

---

## 5. Output encodes (`public/media/`)

| File | Format | Settings | Size |
|------|--------|----------|------|
| `gwyneth-hero.mp4` | H.264 high | **2-pass** `-b:v 2350k`, `gradfun=0.6:15`, `aq-mode=3`, `-g 60`, `+faststart`, no audio | **5.6 MB** |
| `gwyneth-hero.webm` | VP9 | **2-pass** `-b:v 1800k`, `gradfun=0.6:15`, `row-mt`, `-g 60`, no audio | **4.3 MB** |
| `gwyneth-hero-poster.jpg` | JPEG | frame @ ~16.9 s (clean frontal smile), 1920×1080 | 117 KB |
| `gwyneth-headshot.jpg` | JPEG | `IMG_3730` → `crop=3792:4740:224:560` + grade → 960×1200 | 184 KB |
| `gwyneth-social-01.mp4` | H.264 | `161949` 0.2s+8s, `scale=1080:1920` + grade, CRF 21, silent | 4.3 MB |
| `gwyneth-social-02.mp4` | H.264 | `162617` 0.1s+7.6s, `scale=1080:1920` + grade, CRF 21, silent | 9.8 MB |

- **Size targets:** hero mp4 < 6 MB, webm < 5 MB, poster < 300 KB, headshot < 250 KB — all met.
- **2-pass** is used on the hero because grain-free CRF drifted over the 6 MB cap; a fixed
  bitrate guarantees the ceiling. `gradfun` replaces baked-in grain as cheap anti-banding.
- Social verticals are **bonus** assets for her own posting (not referenced by the site).
  Kept silent to avoid shipping unreviewed off-camera audio; ask if audio versions are wanted.

---

## 6. Reproduce / match new footage

1. `ffprobe` the clip — confirm `bt709` (SDR). If HDR, insert the tonemap stage (§7).
2. Pick beats; find crop `Y` by extracting a test frame (eyeline upper third).
3. Apply the **§2 grade string unchanged** so new clips intercut seamlessly.
4. Assemble with 0.35 s `xfade`, encode per §5.

## 7. If a future clip IS HDR (HLG/HDR10)

Prepend this before the grade to map to SDR bt709:

```
zscale=t=linear:npl=100,tonemap=hable:desat=0,zscale=t=bt709:m=bt709:r=tv,format=yuv420p
```
