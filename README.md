# LTM_bilateral

Local tone mapping for DaVinci Resolve as a DCTL, using base/detail decomposition with a
bilateral filter on CIE L* (after Durand & Dorsey, 2002, *Fast Bilateral Filtering for the
Display of High-Dynamic-Range Images*).

```
base   = bilateral(L)
detail = L - base
L'     = anchor + (base - anchor) * contrast + detail * detailGain
```

Compressing only the base layer reduces global dynamic range while the detail layer keeps
local texture and contrast.

## Files

| File | Status | Input / output |
|---|---|---|
| `LocalToneMap_Bilateral_LabIn.dctl` | **Main, tested in Resolve** | Lab in, Lab out (R = L, G = a, B = b) |
| `experimental/LocalToneMap_Bilateral_SLog3.dctl` | Not tested in Resolve | S-Log3 / S-Gamut3 in and out; does its own Lab conversion |
| `docs/DEVELOPMENT_NOTES.md` | | Build errors, Lab encoding findings, hue-shift debugging, ideas |

## Requirements

- DaVinci Resolve **Studio** (the DCTL effect is not available in the free version).
- Source footage: Sony S-Log3 / S-Gamut3 (the Lab version works with any source, as long as
  the conversion to Lab upstream is correct).

## Installation

1. Copy the `.dctl` file into Resolve's LUT folder
   (Windows: `C:\ProgramData\Blackmagic Design\DaVinci Resolve\Support\LUT\`).
   Project Settings → Color Management → Lookup Tables → **Open LUT Folder** also gets you there.
2. **Restart Resolve.** A new DCTL file isn't reliably loaded until Resolve restarts;
   **Update Lists** alone may not make it appear or load properly.
3. Color page → Effects → ResolveFX Color → **DCTL**. Drag it onto a node and choose the file
   from **DCTL List**.

Don't apply it as a LUT (right-click node → LUT). That route has no sliders, and the filter
must read neighbouring pixels, which only works through the DCTL effect.

After editing a file that is already loaded, choose it again from the DCTL List (or click
Update Lists) to recompile. If the changes don't show up, or you added a new or renamed
`.dctl` file, restart Resolve.
Build errors are written to
`%APPDATA%\Blackmagic Design\DaVinci Resolve\Support\logs\ResolveDebug.txt`.

## Node setup (Lab version)

1. Convert to CIE Lab.
2. DCTL effect with `LocalToneMap_Bilateral_LabIn.dctl`.
3. Convert from Lab back to RGB.

The Lab encoding Resolve passes to the DCTL, as measured during testing:

- **L:** 0 to 1 (L* / 100)
- **a, b:** offset so that neutral = **0.5**

The defaults (Channel units = L 0 to 1, Neutral a = Neutral b = 0.5) match this.

## Parameters

All lightness-related values (range sigma, anchor) are in native **L\* units (0–100)**,
whatever the channel encoding. Pixel values are relative to a 1920-wide frame when
*Scale radius to 1920* is on.

| Slider | Default | Range | What it does |
|---|---|---|---|
| **Radius px** | 24 | 1–100 | Half-width of the filter window. Should be at least about 2× the spatial sigma, or the Gaussian gets cut off. A larger radius means a coarser base layer and a slower node. |
| **Sample step px** | 3 | 1–10 | Spacing between samples inside the window. Cost scales with (radius / step)². The base layer is smooth, so 3–4 is usually indistinguishable from 1. Too high gives blocky or aliased artifacts in the base. Raise this before lowering the radius if playback is slow. |
| **Spatial sigma px** | 12 | 0.5–50 | Gaussian falloff with distance. Sets the scale that separates "base" from "detail": anything smaller than about this size ends up in the detail layer and keeps its contrast. Durand & Dorsey suggest about 2% of image width. |
| **Range sigma L** | 15 | 1–100 | Edge stopping, in L\*. Neighbours differing by more than about 2× this contribute little, so edges stay sharp in the base. **Lower:** fewer halos but more risk of banding or gradient reversal near edges. **Higher:** behaves more like a plain blur and brings halos back. 10–20 is a good starting range. |
| **Base contrast** | 0.6 | 0–1.5 | Scale applied to the base around the anchor. 1 = no change, below 1 = compression (the tone mapping), 0 = completely flat base, above 1 = expansion. |
| **Anchor L** | 50 | 0–200 | The L\* value left unchanged by the base compression. L\* 50 is roughly 18% grey. If mid-tones shift brightness when you lower base contrast, set it to your subject's lightness. DCTL can't read frame-wide statistics, so this is fixed rather than computed per frame, which also avoids flicker. |
| **Detail gain** | 1.0 | 0–3 | Multiplier on the detail layer. Above 1 boosts local contrast and texture, but also noise and halos. Below 1 softens. |
| **Chroma follows L** | 0.5 | 0–1 | How much a/b chroma scales with the lightness change, as ((L′+10)/(L+10))^value. 0 = a/b untouched (images get relatively more saturated where darkened). 1 = chroma keeps its ratio to lightness. The +10 offset stops near-black pixels producing huge factors. |
| **Chroma boost limit** | 1.25 | 1–3 | Maximum chroma multiplier. Stops lifted shadows going out of gamut, which causes hue shifts when converting back to RGB. Reductions in chroma aren't limited. |
| **Neutral a / Neutral b** | 0.5 | −1 to 1 | The stored a/b values of a neutral colour. Chroma is scaled around this point so neutrals stay neutral. Resolve stores a/b with a 0.5 offset. Use 0 for unoffset (signed) Lab. |
| **Scale radius to 1920** | on | checkbox | Scales radius, step and spatial sigma by frame width / 1920, so the look is the same at proxy, HD and UHD resolutions. |
| **Channel units** | L 0 to 1 | menu | How L, a and b are stored. *L 0 to 1:* all channels are L\*/100 (Resolve). *L 0 to 100:* native CIE units. |
| **View** | Result | menu | **Result:** the tone-mapped image. **Base grey:** the base layer only (a/b neutral). **Detail grey:** detail added to L\* 50, so flat areas are mid-grey. **Probe ab:** L set to 50 and a/b exaggerated 10× around the neutral point, for finding Neutral a/b by eye. |

### Quick checks

- **Identity:** set Base contrast = 1 and Detail gain = 1. The output should equal the input
  exactly. If it doesn't, the problem is in the Lab conversion around the DCTL.
- **Units:** in *Base grey* view, the correct setting gives a smoothed image whose strong
  edges stay sharp. An evenly blurred image with halos means the range sigma is effectively
  too large (units set to 0–100 on 0–1 data). A nearly unchanged image means it's too small.
- **Neutral point:** in *Probe ab* view, neutral objects should be mid-grey with no tint.

### Suggested starting point

Range sigma 10–20, spatial sigma 12, radius 24–36, step 3, base contrast 0.5–0.7,
anchor 50, detail gain 1.0, chroma follows L 0.5, boost limit 1.25.

## Known limitations

- **CIELAB hue isn't uniform.** Changing L\* at fixed a\*/b\* shifts perceived hue, most of all
  for blues, which drift toward purple. Oklab would be much more stable (see
  `docs/DEVELOPMENT_NOTES.md`).
- **L\* is a cube root, not a log.** The same L\* detail added to a darkened base is a larger
  ratio, so shadows pulled down gain relative texture and highlights pulled down lose some.
- **Halos and gradient reversal** are reduced by the bilateral filter but not removed.
- **Cost.** Brute-force filtering is O((radius / step)²) per pixel.
