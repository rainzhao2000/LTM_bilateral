# Development notes

A record of decisions and findings from building the DCTL.

## Approach

- Goal: local tone mapping by splitting L into base (bilateral-filtered) and detail
  (L − base), compressing the base, and recombining.
- DCTL's full transform mode exposes the frame as textures (`_tex2D(p_TexR, x, y)`), so
  per-pixel neighbourhood filters are possible in a single node.
- DCTL is single-pass with no intermediate buffers and no frame-wide reductions. That rules
  out normalising the base by its per-frame max/min (as in Durand & Dorsey) and in-node
  separable passes. Instead, compression uses a fixed **anchor**.
- **Sparse sampling** (the sample step) keeps large radii affordable, since the base layer is
  smooth.

## Two versions

1. **S-Log3 / S-Gamut3 in and out** (`experimental/`). Does S-Log3 decoding, the
   S-Gamut3 → XYZ (D65) matrix and the Lab conversion itself. Lab white is set to S-Gamut3
   RGB(1,1,1) in XYZ so neutrals give a = b = 0 exactly. The Lab curve's linear segment
   extends below zero so out-of-gamut values stay invertible. The S-Log3 round trip was
   verified in Python. It hasn't been compiled in Resolve, and it doesn't yet have the
   chroma ratio offset, boost limit or neutral sliders from the Lab version.
2. **Lab in and out** (main). Resolve converts to and from Lab; the DCTL only filters L. One
   texture read per sample, so it's much faster.

## Resolve DCTL build errors encountered

| Error | Cause | Fix |
|---|---|---|
| `wrong argument int p_Width in Transform DCTL` / `main DCTL function has wrong ...` | `transform(...)` signature split across two lines; Resolve checks it with its own text parser | Put the whole signature on one line |
| `mismatch enum definitions and display names for DCTLUI_COMBO_BOX`, followed by every UI parameter "undefined" | A combo label contained a comma (`"Detail (grey, +50)"`), so the parser counted one label too many. One bad `DEFINE_UI_PARAMS` line drops all parameters | No commas (and, to be safe, no brackets or symbols) in labels |
| (pre-emptive) | `_max` / `_clamp` on ints may not exist in DCTL | Plain comparisons / ternaries |
| (pre-emptive) | Brackets and `*` in slider labels might confuse the parser | Plain-word labels |

The `nvrtc: warning: Architectures prior to sm_75 are deprecated` line is an NVIDIA
compiler notice, not an error.

## Resolve's Lab encoding (measured)

- **L** arrives as **0 to 1**.
- **a/b** are offset so that **neutral = 0.5**. Confirmed by reading G/B on grey and by
  zero hue shift once Neutral a/b = 0.5.

## Hue-shift debugging

1. **Unbounded chroma scaling.** The original code scaled a/b by L′/L. Lifted near-black
   pixels gave factors around 10× or more, pushing colours out of gamut, and per-channel
   clipping on the way back to RGB shifted hue. Fix: ratio (L′+10)/(L+10), plus a boost limit.
2. **Offset scaled with chroma.** Because neutral is stored at 0.5, multiplying the stored
   a/b also scaled the offset and tinted the image whenever L′ ≠ L (for example when moving
   the anchor). Fix: scale a/b relative to the Neutral a/b point.
3. Remaining, inherent: CIELAB's non-uniform hue (blue → purple).

Diagnostic order that worked: identity check (contrast 1, detail 1) → set Chroma follows L to
0 → read G/B on grey → Probe ab view.

## Ideas / next steps

- **Oklab** instead of CIELAB for much better hue stability; easy to do in the S-Log3
  version's in-script conversion.
- **Linear-RGB ratio:** filter log2(Y), then multiply linear RGB by Y′/Y. Keeps chromaticity
  exactly (Durand & Dorsey's approach). Saturation then changes with brightness.
- **log2(Y) base/detail** so detail is a pure ratio (range sigma roughly 0.4–1 stops).
- **Guided filter** or **local Laplacian** across chained nodes for fewer halos.
- Two chained nodes (horizontal pass, then vertical) as a faster approximate bilateral.
