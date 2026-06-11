# OG Image Strategy

The image that shows up when your link is shared. Often the only piece of your site someone sees before clicking.

## Specs (non-negotiable)

- **1200×630 pixels** — the exact aspect ratio Twitter/X, Facebook, LinkedIn, Slack, Discord, iMessage, Mastodon use for `summary_large_image` / Open Graph "large" cards.
- **PNG format** — universal scraper support. Skip WebP/AVIF (see below).
- **Under 200 KB ideally**, hard cap 1 MB. Bigger images get stripped by Facebook + Slack.
- **Embedded text legible at 30% size** — preview thumbnails in chat apps are tiny. Test by zooming out browser tab.

## File layout

```
public/og/
├── default.png            ← fallback when page doesn't set $ogImage
├── status-default.png     ← fallback for status pages
├── home.png
├── mcp.png
├── privacy.png
├── terms.png
├── cookies.png
└── ...                    ← one per indexable page
```

One PNG per page slug. No locale suffix, no format alternates.

## Why no locale-suffixed variants (`-en`, `-cs`)

Social scrapers (Facebook, LinkedIn, Slack, Twitter, iMessage) crawl URLs **without cookies**. With cookie-based locale (see `locale.md`), Googlebot and every social scraper always see the fallback locale. They never fetch `og:image` with a CS cookie set.

So:
- Layout that emits `og:image: https://yourdomain.com/og/home-cs.png` based on `app()->getLocale()` only ever gets `home-cs.png` for users with the CS cookie.
- Scrapers never have that cookie → always fetch `home-en.png`.
- The `home-cs.png` file is loaded only by humans who already chose Czech — and they're staring at the actual page, not an unfurl.

Net result: `-cs.png` files are dead bytes on disk. Delete them.

If URL-prefixed locales exist (`/cs/...`), the situation flips and per-locale OG images become real. See `locale.md`.

## Why no WebP / AVIF alternates

Most "modern SEO" guides recommend shipping `og:image` triplets:
```html
<meta property="og:image" content="og/home.png">
<meta property="og:image:type" content="image/png">
<meta property="og:image" content="og/home.webp">
<meta property="og:image:type" content="image/webp">
<meta property="og:image" content="og/home.avif">
<meta property="og:image:type" content="image/avif">
```

In practice, every major social platform ignores WebP and AVIF in OG:
- Facebook / Meta scrapers: PNG/JPG/GIF
- LinkedIn: PNG/JPG
- Twitter/X: PNG/JPG/WebP (recent), but PNG is the only reliable choice
- Slack: PNG/JPG
- Discord: PNG/JPG/GIF
- iMessage: PNG/JPG

Shipping the alternates = 3× the files to maintain + 3× the bytes in HTML + sitemap noise for zero benefit. PNG only.

## Required slot pattern

Layout uses `{{ $ogImage }}` with no fallback. Every indexable page sets:

```blade
<x-slot:ogImage>{{ asset('og/<slug>.png') }}</x-slot:ogImage>
```

For Livewire SFC pages with `#[Layout]` attribute:
```php
<?php
new
#[Layout('components.layouts.marketing')]
#[Title('Privacy Policy')]
class extends Component {};
?>

<x-slot:description>...</x-slot:description>
<x-slot:ogImage>{{ asset('og/privacy.png') }}</x-slot:ogImage>

<div>
  ...
</div>
```

If a page forgets, Blade throws `Undefined variable: $ogImage` at render. Bug surfaces in dev/CI before prod.

## Default fallbacks

Two defaults shipped:
- `og/default.png` — for general public pages that haven't designed a custom image yet (404, MCP-app-rendered status page errors)
- `og/status-default.png` — for status pages without custom branding

Pages explicitly opt into a default via slot:
```blade
<x-slot:ogImage>{{ asset('og/default.png') }}</x-slot:ogImage>
```

This is a per-page choice, not a layout fallback. Discipline.

## Design guidelines

- **Brand mark + page title** — primary content. Keep at center, not edge (Twitter sometimes crops 5% off sides).
- **Brand color background** — match `theme-color` value for visual cohesion across browser chrome + unfurl.
- **One concept per image** — features, screenshots, testimonials are too dense at thumb size.
- **Trademark logo or wordmark** in corner — increases brand recall even for tiny previews.
- **Generate from a template** — Figma file with text variables + locked layers. Bulk export → batch optimize → commit.

## Optimization pipeline

1. Export PNG at 1200×630 from Figma/design tool.
2. Run through `pngquant` or `oxipng` to lossy-compress (~85% quality is invisible at OG size, ~40% file size).
3. Final size target: 50–150 KB.

```bash
oxipng -o max og/*.png
# OR
pngquant --quality=85-95 --force --ext .png og/*.png
```

## Disk hygiene

Audit periodically:
```bash
# orphaned files (no page references them)
ls public/og/ | while read f; do
  slug="${f%.png}"
  if ! grep -rq "og/$slug" resources/views/ app/; then
    echo "orphan: og/$f"
  fi
done
```

Delete orphaned OG images on the same commit as the page that stopped using them.
