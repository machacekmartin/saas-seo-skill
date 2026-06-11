# Open Graph + Twitter Cards

Goal: useful unfurls on Slack, Discord, X, LinkedIn, iMessage with minimum tag bloat. Most "complete" OG implementations ship 3× the tags real consumers actually read.

## The block (indexable layouts only)

```html
<meta property="og:site_name" content="{{ config('app.name') }}">
<meta property="og:type" content="website">
<meta property="og:url" content="{{ url()->current() }}">
<meta property="og:title" content="{{ $title }}">
<meta property="og:description" content="{{ $description }}">

<meta property="og:image" content="{{ $ogImage }}">
<meta property="og:image:type" content="image/png">
<meta property="og:image:width" content="1200">
<meta property="og:image:height" content="630">
<meta property="og:image:alt" content="{{ $title }}">

<meta property="og:locale" content="en_US">
```

That's it. No alternate locales, no webp/avif variants, no `og:image:secure_url`, no `og:see_also`, no `article:*`.

## Why each field

| Tag | Purpose |
|---|---|
| `og:site_name` | Small brand label above title in unfurls. Branding, NOT page title. |
| `og:type` | `website` for landing/marketing, `article` for blog/legal/docs (prop-driven if mixed layout). |
| `og:url` | Canonical for share attribution. Match `<link rel="canonical">`. |
| `og:title` | Bold headline in unfurl. Page title, not site title. |
| `og:description` | Body text below title. |
| `og:image` | 1200×630 PNG. The hero of the unfurl. |
| `og:image:width/height` | Lets scrapers pre-allocate layout (Slack, LinkedIn). |
| `og:image:alt` | Accessibility + AI alt-text. Reuse `$title`. |
| `og:locale` | `en_US` etc. Hardcode if single-locale indexing. |

## What NOT to ship

### `og:image:secure_url`
Deprecated. Only useful when serving mixed-content pages — irrelevant on HTTPS-only sites.

### WebP / AVIF `og:image` alternates
Almost no social scraper picks them up:
- Facebook / LinkedIn / Slack — PNG/JPG only
- Twitter/X — PNG/JPG only
- iMessage — PNG/JPG only

Shipping webp/avif variants = dead bytes in HTML + dead files on disk + sitemap noise. PNG only.

### `og:locale:alternate`
Same problem as `hreflang` — only valid if each locale has a distinct URL the scraper can fetch. Cookie/session locales = lie.

### `og:see_also`
Niche field. Facebook used to consider it for related content recommendations; it's effectively dead. Footer links serve the same discovery purpose.

### `article:*` tags
- `article:published_time`, `article:modified_time` — covered better by JSON-LD `Article.datePublished` / `dateModified`
- `article:author` — covered by JSON-LD `Article.author`
- `article:section` — low signal

If users need to see "Last updated: 2026-06-10" on legal docs, render it as visible body text.

## Twitter Cards

```html
<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:title" content="{{ $title }}">
<meta name="twitter:description" content="{{ $description }}">
<meta name="twitter:image" content="{{ $ogImage }}">
<meta name="twitter:image:alt" content="{{ $title }}">
```

### Card types

- `summary_large_image` — 1200×630 big horizontal image above headline. Use this when you have OG-spec imagery.
- `summary` — 160×160 tiny square next to text. Only for sites without landscape imagery.
- `player`, `app` — irrelevant for most SaaS.

### `twitter:site` + `twitter:creator`

```html
<meta name="twitter:site" content="@yourbrand">
<meta name="twitter:creator" content="@yourbrand">
```

**Only ship if the account exists.** Pointing to a dead handle = broken Twitter card attribution + Mastodon rel=me verification fails.

If account doesn't exist yet, drop these two lines and write a TODO plan file to re-add when account is created. Keep `twitter:card` — X still uses it to pick card layout via OG fallback.

### Twitter falls back to OG

X reads `og:title`, `og:description`, `og:image` when `twitter:*` is missing. The duplicate `twitter:title/description/image` block is technically redundant. Reasons to keep:
- Some platforms (X embeds, Telegram) treat them differently from OG.
- Belt-and-suspenders for consumers that prioritize twitter:* over og:*.

Cost is 4 lines. Keep.

## OG image required slot

Layout uses `{{ $ogImage }}` with no fallback. Every indexable page provides:

```blade
<x-slot:ogImage>{{ asset('og/<page-slug>.png') }}</x-slot:ogImage>
```

If page forgets, Blade throws at render. Catch it in CI/local before prod.

## OG image strategy

See `og-images.md`. TL;DR: 1200×630 PNG, one per page slug at `public/og/<slug>.png`. No locale suffix (scrapers have no cookie). One default fallback at `og/default.png`. Status pages get their own at `og/status-default.png`.
