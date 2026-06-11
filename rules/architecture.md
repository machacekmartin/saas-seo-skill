# Layout Architecture

Most SaaS apps have 3–4 distinct rendering contexts. Each has different SEO needs. Mixing them into one mega-layout is the source of most SEO cargo cult.

## The four-layout pattern

| Layout | Pages it serves | Indexable | SEO meta |
|---|---|---|---|
| **marketing** | Home, landing, pricing | Yes | Full (OG, Twitter, JSON-LD, canonical, rel=me, webmaster verify) |
| **public** | Legal docs, MCP docs, status pages, 404 | Mixed (controlled by `$noindex` prop) | Full meta but `og:type` is a prop (`'website'` default, `'article'` for legal) |
| **app** | Dashboard, settings, monitor detail, billing | No | Stripped to UX-only |
| **guest** | Auth screens (login, register, forgot password) | No | Stripped to UX-only |

The decision is binary at the layout level — don't try to conditionally emit OG tags based on per-page flags. If the layout's pages are behind a login wall, they get the stripped meta.

## What "indexable" layouts emit

In `<head>`:

- `<meta charset>`, `<meta viewport>`, `<meta csrf-token>` — required infrastructure
- `<title>{{ $title }}</title>` — required slot, no fallback
- `<meta name="description" content="{{ $description }}">` — required slot
- `<meta name="application-name" content="{{ config('app.name') }}">`
- `<meta name="robots" content="index,follow">` (or dynamic via `$noindex` prop)
- `<meta name="googlebot" content="..., max-snippet:-1, max-image-preview:large, max-video-preview:-1">`
- `<link rel="canonical" href="{{ url()->current() }}">`
- Theme + UI block: theme-color matched to brand canvas, color-scheme, apple-mobile-web-app-*, format-detection, mobile-web-app-capable
- Favicon set: svg + ico + apple-touch-icon + manifest
- Identity: `<link rel="me" href="https://github.com/<real-handle>">`, `<link rel="author" href="https://<domain>/#organization">` (matches JSON-LD `@id`)
- Open Graph block: site_name, type, url, title, description, single PNG image (no webp/avif alternates), og:locale hardcoded
- Twitter card: `summary_large_image` + title/description/image/image:alt (drop `twitter:site` / `twitter:creator` if no real X handle)
- DNS prefetch / preconnect for actual third parties (fonts CDN) — NOT for `rel=me` targets you never fetch
- Webmaster verification (`@if`-gated on env vars)
- JSON-LD: `<x-seo.json-ld :schemas="['organization', 'website', 'softwareApplication', ...]">`
- `@vite([...])`, `@livewireStyles` (infra)

## What "behind-login" layouts emit

In `<head>`:

- charset, viewport, csrf-token
- `<title>{{ $title }}</title>`
- `<meta name="robots" content="noindex,nofollow">`
- Theme + UI block (same as indexable — these are UX, not SEO)
- Favicon set
- Preconnect for fonts CDN
- @vite, @livewireStyles

That's it. No description, no OG, no Twitter, no canonical, no rel=me, no JSON-LD, no webmaster verification, no generator.

## Why the split matters

- **Smaller HTML payload** behind login (every byte counts on the app pages users hit constantly).
- **Less coupling** — changes to OG strategy don't touch dashboard layout.
- **Force discipline at page level** — every indexable page must set `$title`, `$description`, `$ogImage`. Behind-login pages just set `$title`. Discoverability of missing slots is automatic (Blade throws).
- **No accidental indexing leak** — even if `/dashboard` URL leaks via referer, `noindex,nofollow` keeps it out of SERP. Zero OG = useless Slack unfurl, which is correct.

## Mixed-indexability layouts

The `public` layout is the tricky one — serves both indexable legal/docs pages AND noindex status admin pages.

Pattern: prop-driven robots.

```blade
@props(['noindex' => false, 'ogType' => 'website'])
...
<meta name="robots" content="{{ $noindex ? 'noindex,nofollow' : 'index,follow' }}">
<meta property="og:type" content="{{ $ogType }}">
```

Default props match the most common use (indexable, website). Status admin pages opt in with `:noindex="true"`.

## Anti-patterns to remove

- Single layout that conditionally emits OG tags based on `auth()->check()` — fragile, hides intent.
- Different page templates passing different `$pageType` flags to one layout. The layout should know its bucket.
- `app.blade.php` (used everywhere) with the full SEO meta block — wasteful + leaks meta into pages that shouldn't have any.
