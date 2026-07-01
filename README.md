# Logo Tile Optimizer — Engineering Notes

A single-file, browser-based utility that takes any logo and automatically produces a centered, correctly scaled, shape-optimized export for Circle, Oval, Square, and Rectangle tile placeholders — no manual resizing, repositioning, or background cleanup. Everything runs client-side: nothing is uploaded to a server, which also means there's no backend to host or maintain.

## Running it

Open `logo-tile-optimizer.html` directly in any modern browser (Chrome, Firefox, Safari, Edge). No build step, no install, no server. Double-click the file or drag it into a browser tab.

## How the engine maps to the spec

### 1. Detect logo
`findContentBBox()` scans pixel data and unions every region of "real content" into one bounding box — alpha-based for transparent sources, color-distance-from-background for opaque ones. It correctly handles logos made of disjoint elements (icon + wordmark with a gap between them, a trademark dot floating away from the mark) by taking the union of all content, not just the largest connected blob, since a real logo's icon and wordmark need to scale and center as one composition. For performance on very large sources, detection runs on a downscaled analysis copy (capped at 1100px on the long edge) and the result is mapped back to full-resolution coordinates — this keeps a 4000×4000px upload processing in about a second instead of freezing the tab on 16 million pixels, at the cost of a few pixels of quantization on extreme sources (sub-1% of content size, and tunable via the analysis cap in code if a project needs tighter precision than speed).

### 2. Auto scale
`computeLayout()` always scales by a single uniform factor applied to both width and height (`scale = containScale × fillRatio`), so distortion is structurally impossible, not just visually unlikely. Fill ratio defaults to 66% of the shape's safe area and is exposed as a slider (40–85%) for cases where a brand wants tighter or looser framing.

### 3. Auto align
Every shape draws through the same centering math (`destX/destY = (frame − content)/2`), so centering is identical regardless of shape — verified to sub-half-percent accuracy in testing.

### 4. Background handling
On upload, the tool checks for alpha transparency. If the source is opaque, it samples the border pixels (mode color, not a naive corner pixel, so it's resilient to a stray watermark or jpeg artifact sitting in one corner) and uses that as the detected background, with a one-click override picker. If the source is transparent, the person is prompted right in the Background panel to keep transparency, pick a color, or type a HEX/RGB value — matching the spec's three options exactly. The chosen color extends seamlessly behind the logo at every frame size, including for JPEG export (which can't carry transparency, so the canvas is flattened onto the chosen color before encoding rather than silently turning transparent pixels black, which is what naive JPEG export usually does).

### 5. Shape optimization
`getSafeArea()` computes a distinct usable region per shape rather than reusing one rectangle for everything:
- **Circle** — inset to the square inscribed inside the circle, so corner content can never be clipped by the circular mask.
- **Oval** — same inscribed-rectangle logic against the ellipse, which naturally produces more conservative vertical margins than horizontal — appropriate for a wide frame.
- **Square / Rectangle** — uniform percentage padding on all sides, with rectangle's wider frame giving it more usable horizontal room automatically (no special-casing needed, since margin is a percentage of the shorter side).

## Edge cases (from the original spec)

| Case | Handling |
|---|---|
| Very wide / very tall logos | Uniform-scale fit against the safe area on whichever axis is tighter; verified down to a 13.75:1 wordmark and up to a 1:13.75 vertical mark with zero clipping. |
| Tiny logos | Upscaled (scale factor can exceed 1×) using high-quality canvas smoothing rather than left undersized. |
| Excessive whitespace | Boundary detection trims to real content first, so a 100px mark buried in a 2000px transparent canvas is treated as a 100px logo, not rendered as a speck. |
| Transparent logos | Three-way prompt (keep / pick / hex), as specified. |
| Logos with colored backgrounds | Border-sampled dominant color, extended seamlessly on canvas expansion. |
| Vector (SVG) logos | Rasterized at a generous working resolution (long edge clamped to 800–2200px) before the same pipeline runs, so vector logos get equally sharp output without a separate code path. |
| High-resolution raster images | Analysis runs on a capped-resolution copy for speed; final composition draws from the full-resolution source, so output sharpness isn't reduced. |
| Low-resolution images requiring upscaling | Same as "tiny logos" above. |

## Success criteria — verification approach

Rather than eyeballing it, this build was checked with an automated browser test suite (91 assertions across three test files) that drives the real app — not a mock — through file upload, shape switching, background-mode switching, custom sizing, and downloads, then inspects actual rendered pixel data and the underlying float layout math. Specifically verified: zero distortion (scale factor is mathematically identical on both axes, not just visually close), the 66% fill target is hit to within 0.1% on every shape, centering holds to within 0.4% of frame center, content never exceeds the shape's safe area, and downloaded files have the right dimensions, color mode, and alpha handling for each export format.

## Future enhancements — status against the original roadmap

- **AI-powered logo boundary detection** — not built. Current detection is deterministic (alpha/color-distance based), which is fast, free, fully private, and works well for clean logos, but won't separate a logo from a busy photographic background. A real version of this needs a vision model call, which means a backend — out of scope for a client-only tool.
- **Automatic background removal for non-transparent images** — not built, same constraint as above (this is a segmentation problem, not a boundary-trim problem).
- **Smart dominant color extraction with gradient support** — partially built. Dominant-color extraction works (mode color from border pixels); gradient backgrounds aren't detected as gradients and would be approximated as a single flat color.
- **Optical centering** — not built. Current centering is geometric (bounding-box center), which is correct and predictable for symmetric marks but won't compensate for a logo that's visually lopsided (e.g., a check-mark icon that "looks" off-center despite a centered bounding box).
- **Batch processing** — not built. Architecture supports it reasonably easily (the render pipeline is already a pure function of `(sourceCanvas, bbox, shape, frame, background)`), but the UI is single-logo only.
- **Export presets for different portal requirements** — partially built via the preset size chips per shape; there's no saved/named preset system for a specific portal's full requirement set.
- **Preview before download** — built. The proof view and the four-shape proof sheet are live previews; nothing downloads until the person clicks Download.

## Extending it

- **New frame size**: add an entry to the `PRESETS` object for the relevant shape.
- **New placeholder shape**: add a case to `getSafeArea()` defining its usable region, a default margin in `SHAPE_DEFAULT_MARGIN`, and a button in the shape grid — the scale/align/render pipeline is shape-agnostic and needs no changes.
- **Tighter boundary-detection precision on huge sources**: raise the `1100` cap in `buildAnalysisCanvas()` — trades speed for precision.

## Known limitations

Backgrounds that are gradients, photographic, or otherwise non-solid won't be detected or removed — the tool will either treat a transparent source correctly or treat an opaque source's border color as flat, which can look wrong against a busy background. The honest fix for that is a background-removal model, not a bigger threshold.
