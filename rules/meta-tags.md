# Meta Tags — Keep / Drop Catalog

Per-tag verdict with reasoning. The default verdict is **drop** unless the tag has a real consumer for *your* site.

## Required infrastructure (always keep)

| Tag | Why |
|---|---|
| `<meta charset="utf-8">` | Required first thing in `<head>` for byte parsing. |
| `<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">` | Mobile rendering + iPhone notch handling. |
| `<meta name="csrf-token" content="{{ csrf_token() }}">` | Livewire / Axios read for `X-CSRF-TOKEN` header. |

## Document basics (keep on indexable layouts)

| Tag | Verdict | Why |
|---|---|---|
| `<title>{{ $title }}</title>` | Keep — required slot | Browser tab + SERP. No fallback; force pages to provide. |
| `<meta name="description" content="{{ $description }}">` | Keep on indexable / drop on noindex | SERP snippet. No fallback. Drop entirely from behind-login layouts. |
| `<meta name="application-name" content="{{ config('app.name') }}">` | Keep | Windows tile / OS app integration. |
| `<meta name="generator" content="Laravel">` | **Drop** | Info leak, no SEO value. Tells attackers your stack. |
| `<meta name="robots" content="index,follow">` | Keep | Crawl + index hint. Use `noindex,nofollow` for app/guest/private. |
| `<meta name="googlebot" content="index,follow, max-snippet:-1, max-image-preview:large, max-video-preview:-1">` | Keep | Unlocks unlimited SERP snippet + large image preview. |
| `<meta http-equiv="content-language" content="...">` | **Drop** | Superseded by `<html lang>`. Legacy. |

## Theme + UI (UX meta — keep on all layouts)

| Tag | Verdict | Why |
|---|---|---|
| `<meta name="theme-color" content="#f6f4ec">` | Keep | Mobile browser chrome color. Use actual brand canvas hex, not pure white. |
| Dark `theme-color` with `media="(prefers-color-scheme: dark)"` | Conditional | **Drop if site has no dark mode** — dead code. Only emit if site palette has dark variant. |
| `<meta name="color-scheme" content="light">` | Keep | Hints form control rendering. Use `"light"` not `"light dark"` unless dark mode exists. |
| `<meta name="apple-mobile-web-app-title" content="{{ config('app.name') }}">` | Keep | iOS home screen label. |
| `<meta name="apple-mobile-web-app-capable" content="yes">` | Keep | iOS standalone PWA mode. |
| `<meta name="apple-mobile-web-app-status-bar-style" content="default">` | Keep | iOS status bar appearance. |
| `<meta name="format-detection" content="telephone=no">` | Keep | Stops iOS auto-linking "8.5.21" → `tel:`. Important for apps showing version/metric numbers. |
| `<meta name="mobile-web-app-capable" content="yes">` | Keep | Android Chrome PWA standalone. |

## Identity + icons

| Tag | Verdict | Why |
|---|---|---|
| `<link rel="icon" href="/favicon.svg" type="image/svg+xml">` | Keep | Modern browsers prefer SVG (sharp at any DPR). |
| `<link rel="icon" href="/favicon.ico" sizes="32x32">` | Keep | Fallback for older browsers + `/favicon.ico` default fetch. |
| `<link rel="apple-touch-icon" href="/apple-touch-icon.png">` | Keep | iOS home screen icon (180×180). Without it, iOS screenshots the page. |
| `<link rel="mask-icon" href="/favicon.svg" color="#XXX">` | **Drop** | Legacy Safari pinned-tab icon. Needs monochrome silhouette SVG anyway. Safari 12+ uses `rel=icon` SVG. |
| `<link rel="manifest" href="/manifest.webmanifest">` | Keep | PWA manifest. Required for Android Chrome install prompt. |
| `<link rel="search" type="application/opensearchdescription+xml" ...>` | **Drop** | Only useful if site has real `?q=` search. Most SaaS sites don't. |

## Identity verification + author

| Tag | Verdict | Why |
|---|---|---|
| `<link rel="me" href="https://github.com/<real-handle>">` | Keep only if handle real | Mastodon / IndieAuth verification. Points to YOUR account. Drop if account doesn't exist yet. |
| `<link rel="me" href="https://x.com/<handle>">` | Drop until account exists | Pointing to a dead handle = broken signal. Track as a TODO; re-add when account live. |
| `<link rel="author" href="https://<domain>/#organization">` | Keep | Anchor must match JSON-LD `@id` of Organization node. |

## Canonical + locale

| Tag | Verdict | Why |
|---|---|---|
| `<link rel="canonical" href="{{ url()->current() }}">` | Keep on indexable, drop behind login | Consolidates duplicate URLs. Bare `url()->current()` — no `?locale=X` parameter unless URL-prefixed locale routes exist. |
| `<link rel="alternate" hreflang="...">` | Drop unless distinct per-locale URLs exist | If locale is cookie/session-based, hreflang is a lie. See `locale.md`. |

## Open Graph + Twitter (see `open-graph.md`)

Strip from behind-login. Single PNG only. Drop `og:image:secure_url`, `og:see_also`, `og:locale:alternate`. Hardcode `og:locale="en_US"` if EN-only. See full breakdown in `open-graph.md`.

## Perf hints

| Tag | Verdict | Why |
|---|---|---|
| `<link rel="preconnect" href="https://fonts.bunny.net" crossorigin>` | Keep | TCP+TLS warmup for font CDN = faster first font byte. |
| `<link rel="dns-prefetch" href="https://fonts.bunny.net">` | Keep | DNS resolution fallback for browsers without preconnect support. |
| `<link rel="dns-prefetch" href="https://github.com">` (for rel=me) | **Drop** | No actual fetch happens to rel=me targets on page load. Wasted DNS lookup. |

## Webmaster verification (config-driven)

| Tag | Verdict |
|---|---|
| `<meta name="google-site-verification">` | Keep, env-gated |
| `<meta name="msvalidate.01">` (Bing) | Keep, env-gated |
| `<meta name="yandex-verification">` | Keep, env-gated (drop in template if no RU/Russophone audience) |
| `<meta name="p:domain_verify">` (Pinterest) | Keep, env-gated (drop in template if not a visual brand) |
| `<meta name="facebook-domain-verification">` | Keep, env-gated (drop in template if not running Meta ads) |

Pattern:
```blade
@if (! empty(config('seo.verification.google')))
    <meta name="google-site-verification" content="{{ config('seo.verification.google') }}">
@endif
```

With `config/seo.php`:
```php
return [
    'verification' => [
        'google' => env('SEO_VERIFY_GOOGLE'),
        'bing' => env('SEO_VERIFY_BING'),
        // ...
    ],
];
```

## Misc to actively drop

| Tag | Why drop |
|---|---|
| `@stack('head')` | Only keep if pages actually `@push('head')`. Otherwise dead code. |
| `<meta property="og:see_also">` | Niche OG field, few consumers, weak signal. |
| `<meta property="og:locale:alternate">` | Same locale-URL problem as hreflang. |
| `<meta property="og:image:secure_url">` | Deprecated. HTTPS-only sites don't need it. |
| `<meta property="og:image" content="<webp_url>">` and `avif` variants | Most social scrapers ignore webp/avif. Single PNG only. |
| `<meta property="article:author">`, `article:section` | Low signal. JSON-LD Article schema is the better channel. |
| `<meta property="article:published_time">`, `article:modified_time` | If you want dated docs, use visible body text + JSON-LD `datePublished` / `dateModified`. |

## Required slots (no fallbacks)

For indexable layouts, force pages to fill:
- `$title`
- `$description`
- `$ogImage`

In Blade: `{{ $title }}` — no `?? 'default'`. Missing slot throws `Undefined variable: $title` at render. That's the feature.

Pages set via `<x-slot:title>Page title</x-slot:title>` blocks, or Livewire's `#[Title('...')]` attribute (which also fills the `$title` slot in the layout component).

For behind-login layouts, only `$title` is required. Drop description + ogImage slots from pages using app/guest layouts.
