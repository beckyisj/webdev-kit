---
name: og-image
description: Generate an OG image (1200x630) for any website or tool. Designs for small-preview readability (WhatsApp, Slack, iMessage). Use when creating or updating social preview images.
user-invocable: true
allowed-tools: Read, Write, Edit, Bash, Glob
argument-hint: [tool-name]
---

# OG Image Generator

Generate a branded OG/social preview image for any website or tool.

**Argument**: `$ARGUMENTS` is the tool/site name (e.g., "Podcast Packager", "PRISM").

## Design Principles

**OG images appear tiny** — WhatsApp, Slack, iMessage, and Twitter show them as small thumbnails. Design for readability at thumbnail size:

- **Fill the frame**: Use minimal padding. Content should occupy 70-80% of the canvas, not float in whitespace.
- **Large text**: Title text should be at least 60-80px in the 1200x1200 canvas. Subtitles at least 28-34px.
- **Bold contrast**: Text must be clearly readable against the background at small sizes.
- **Few elements**: Logo + title + one subtitle + domain. No small pills or fine details that disappear at thumbnail size.
- **Test mentally**: Imagine the image at 300x157px — can you still read the title?

## Steps

### 1. Check for existing brand assets
Look for existing logos, color schemes, or SVGs in the project to match the brand.

For YouTube Producer tools specifically, read the master template at `~/Cursor/youtube-producer/youtubeproducer-app/og-image.svg`.

### 2. Create the SVG
Create a new SVG at the target project's `public/og-image.svg` (or project root if no `public/` dir).

**Canvas & crop zone:**
- **viewBox**: `0 0 1200 1200` (square canvas)
- **Crop zone**: y=285 to y=915 — this 630px band is what becomes the final 1200x630 image
- **All visible content** must be within this crop zone, vertically centered
- **Tight layout**: Content should fill most of the crop zone. Aim for ~60px top/bottom padding within the zone, not 150px+

**Content hierarchy (top to bottom within crop zone):**
1. **Logo/icon**: Large and prominent (scale to ~120-160px tall)
2. **Title**: 60-80px, font-weight 700, high contrast fill
3. **Subtitle**: 28-34px, slightly muted fill
4. **Domain**: 24-28px at bottom of content zone, muted fill

**Typography:**
- Font: `-apple-system, 'Helvetica Neue', sans-serif` (or project-specific fonts)
- All text: `text-anchor="middle"` at `x="600"`
- CJK text: Use `'PingFang HK', 'Noto Sans TC'` as primary font-family

**YouTube Producer tools specifically:**
- Background: stone gradient (`#fafaf9` → `#f0efed`, diagonal)
- Accent: teal gradient bar at y=285
- Teal glow ellipse, teal-outlined pills
- Domain: `youtubeproducer.app`

### 3. Render to PNG
Run these commands from the project directory:

```bash
qlmanage -t -s 2400 -o . og-image.svg 2>/dev/null
sips --cropToHeightWidth 1260 2400 og-image.svg.png
sips -z 630 1200 og-image.svg.png
mv og-image.svg.png public/og-image.png  # or project root
```

If the SVG is in `public/`, adjust paths:
```bash
cd public
qlmanage -t -s 2400 -o . og-image.svg 2>/dev/null
sips --cropToHeightWidth 1260 2400 og-image.svg.png
sips -z 630 1200 og-image.svg.png
mv og-image.svg.png og-image.png
```

### 4. Verify
Read the generated PNG to visually verify:
- Is the content filling the frame (not floating in whitespace)?
- Would the title be readable at 300px wide?
- Are all elements within the crop zone?

### 5. Remind about cache purging
After deploying, remind the user to purge OG caches:
- LinkedIn Post Inspector
- Facebook Sharing Debugger
- Twitter Card Validator

## Rules
- NEVER use PIL/Pillow bitmap generation — SVG + qlmanage/sips only
- Content MUST be centered in the y=285–915 crop zone
- **SIZE UP**: When in doubt, make elements bigger. Whitespace is the enemy of OG images.
- Verify text symbols ($, S, etc.) aren't mirrored in hand-drawn SVG paths
