---
name: svg-to-png
description: Convert SVG files to PNG using rsvg-convert from librsvg. Use this skill whenever the user asks to convert an SVG to PNG, regenerate a PNG from an SVG, render an SVG, export an SVG as PNG, or update/regenerate the OG image. Also triggers on mentions of "rsvg-convert", "svg to png", "regenerate png", or "og-image.png".
---

# SVG to PNG Conversion

Use `rsvg-convert` (from the `librsvg` Homebrew package) for high-quality SVG to PNG conversion. This produces significantly better results than headless Chrome screenshots.

## Prerequisites

If `rsvg-convert` is not installed:

```bash
brew install librsvg
```

## Usage

```bash
rsvg-convert -w <width> -h <height> <input.svg> -o <output.png>
```

- `-w` — output width in pixels
- `-h` — output height in pixels
- Read the SVG's `viewBox` or `width`/`height` attributes to determine the correct output dimensions
- If only one dimension is specified, the other scales proportionally

## Project-specific: OG Image

The moly project has an OG image workflow:

1. Source SVG: `mascot/og-image-wrapper.svg` (1200x630)
2. Output PNG: `mascot/og-image.png`
3. Command:
   ```bash
   rsvg-convert -w 1200 -h 630 mascot/og-image-wrapper.svg -o mascot/og-image.png
   ```

Always regenerate `og-image.png` after editing `og-image-wrapper.svg`.
