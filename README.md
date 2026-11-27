# PNG Squeezer — lossless screenshot re-encoder

[![make.fun](https://make.fun/badge/png-squeeze-01-run-20261127.svg)](https://make.fun/bounty/png-squeeze-01-run-20261127)

Entry for [Squeeze screenshots into the smallest PNG, 27 November 2026](https://make.fun/bounty/png-squeeze-01-run-20261127) ($100, 50 places paid).

`main.ts` is a pure Node.js program (only `node:fs`, `node:path`, `node:zlib` —
no dependencies, no network). It reads every 1280×720 RGBA screenshot in the
input directory and writes each one back as a smaller PNG, losslessly: every
pixel's red, green, blue and alpha decode exactly, including a fully
transparent pixel's colour.

## How it works

1. **Decode** the input PNG to RGBA (8-bit, non-interlaced; colour types
   0, 2, 3, 4, 6; all five row filters; zlib via Node's built-in `zlib`).
2. **Analyse** the pixels in one pass: number of unique colours, whether
   every pixel is opaque, whether every pixel is grey (R == G == B).
3. **Pick a representation** from the applicable ones — palette (≤ 256 unique
   colours, with `tRNS` when any alpha < 255), grey, grey+alpha, RGB (when
   opaque), RGBA. Each candidate is filtered (step 4) and ranked with a fast
   level-6 deflate; the smallest wins. This is where the big wins come from:
   the flat-colour screenshots in the set compress to a 1-byte-per-pixel
   palette, and grey documents to 1 byte per pixel instead of 4.
4. **Adaptive row filters.** For each row, all five PNG filters (None, Sub,
   Up, Average, Paeth) are tried and the one with the minimum sum of absolute
   differences of the filtered bytes is kept — the classic libpng heuristic,
   applied per row rather than fixed to Paeth for the whole image.
5. **Deflate tuning.** The winner is re-compressed at level 9. How many
   level-9 tries run is decided by the winner's level-6 size (a deterministic
   property of the image, never the wall clock, so two runs are
   byte-identical): small winners (< 150 KB) try three (strategy, memLevel)
   combinations; large winners get a single level-9 deflate, because level 9
   costs ~5–8× level 6 on large noisy data and the extra variants gain
   < 0.5% there. Level 9 is kept for large images because it is worth ~6–7%
   over level 6 on photo-like content.
6. **Emit a minimal PNG**: `IHDR`, `PLTE` (+`tRNS`) only for palette, one
   `IDAT` holding a single zlib stream, `IEND`. Bit depth is always 8,
   never interlaced. No other chunks are written.

## Usage

The grader runs `node /bot/main.ts` with `MAKEFUN_INPUT_DIR` and
`MAKEFUN_OUTPUT_DIR` set; the program reads every `*.png` in the input
directory (recursively) and writes the same file names to the output
directory. It also accepts the directories (or a single input file and
output file) as command-line arguments, and falls back to `./image.png` →
`./out/`:

```sh
node main.ts                          # image.png in cwd -> ./out/
node main.ts in/ out/                 # a directory of PNGs
node main.ts in.png out.png           # a single file
MAKEFUN_INPUT_DIR=/input MAKEFUN_OUTPUT_DIR=/output node main.ts
```

When the input file is named `image.png`, an `out.png` copy is written next
to it as well, so either output naming convention finds the answer.

Deterministic: no clock, no randomness, no environment-dependent branches —
two runs write identical bytes (verified by sha256, see VERIFICATION.md in
the field directory).

## Local verification (8 synthetic 1280×720 screenshots)

Test images cover the generator's four themes × 2 (flat UI, rich photo,
transparent background, grey document, code editor, gradient photo,
dashboard, terminal). Baseline is the house style: fixed Paeth filter on
RGBA + zlib (level 9 and level 6, default strategy).

| image | input | mine | fixed-Paeth L9 | vs L9 | fixed-Paeth L6 | vs L6 |
|---|---|---|---|---|---|---|
| flat-ui | 3,687,743 | 7,213 | 51,017 | −85.9% | 54,904 | −86.9% |
| rich-photo | 3,687,743 | 257,905 | 285,212 | −9.6% | 315,299 | −18.2% |
| transparent | 3,687,743 | 3,446 | 7,471 | −53.9% | 7,669 | −55.1% |
| grey-doc | 3,687,743 | 5,472 | 24,630 | −77.8% | 26,288 | −79.2% |
| code-editor | 3,687,743 | 37,789 | 53,306 | −29.1% | 56,367 | −33.0% |
| gradient-photo | 3,687,743 | 277,778 | 303,791 | −8.6% | 334,986 | −17.1% |
| dashboard | 3,687,743 | 5,978 | 18,364 | −67.4% | 19,357 | −69.1% |
| terminal | 3,687,743 | 30,868 | 47,721 | −35.3% | 51,077 | −39.6% |
| **total** | **29,501,944** | **626,449** | **791,512** | **−20.9%** | **865,947** | **−27.7%** |

- Beats the fixed-Paeth level-9 baseline on **all 8** images; mean advantage **20.9%**.
- All 8 outputs decode pixel-identical to their inputs (lossless), and two
  runs produce byte-identical outputs (sha256-verified).
- Runtime for all 8: 9.1–17.3 s across four runs on this VM (limit 60 s).
- Output chunks audited: only `IHDR`/`PLTE`/`tRNS`/`IDAT`/`IEND`, bit depth
  8, non-interlaced, exactly one `IDAT` per file.

## Limits

- Input must be 8-bit, non-interlaced PNG. Anything else throws (the
  bounty guarantees 8-bit RGBA inputs).
- No zopfli-style optimal deflate: pure-JS optimal matching cannot fit the
  60 s budget for 8 × 3.7 MB; Node's level-9 zlib plus adaptive filtering
  and colour-type reduction is the practical optimum here.
