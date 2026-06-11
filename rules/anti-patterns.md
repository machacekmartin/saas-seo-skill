# Anti-Patterns — Remove on Sight

The most common SEO cargo cult shipped by scaffolds + "comprehensive SEO setup" tutorials. Each entry below: what it looks like + why it's harmful + what to do instead.

## 1. Heavy `@php` block computing SEO variables in layout

```php
@php
    $appName = config('app.name', 'Notdown');
    $titleStr = trim((string) ($title ?? ''));
    $descriptionStr = trim((string) ($description ?? '')) ?: __('pages.home.meta.description');
    $ogImageStr = trim((string) ($ogImage ?? '')) ?: asset('og/default.png');
    $fullTitle = $titleStr && ! str_contains($titleStr, $appName)
        ? $titleStr.' · '.$appName
        : ($titleStr ?: $appName);
    $resolvedOgTitle = $titleStr ?: $appName;
    $resolvedOgAlt = $resolvedOgTitle;
    $currentUrl = url()->current();
    $defaultLocale = config('app.fallback_locale', 'en');
    $locale = app()->getLocale();
    // ... 40 more lines
@endphp
```

**Why bad:** Hides intent. Magic resolution. Pages can't tell what the layout will emit without reading the @php block. Bugs surface only at render of edge cases.

**Fix:** Inline expressions at usage sites. Require pages to fill slots:
```html
<title>{{ $title }}</title>
<meta name="description" content="{{ $description }}">
<meta property="og:image" content="{{ $ogImage }}">
```

No fallbacks. Pages MUST set `$title`, `$description`, `$ogImage`. Missing slot throws.

## 2. Fallback chains for required meta

```html
<title>{{ $title ?? config('app.name', 'Notdown') }}</title>
```

**Why bad:** Hides missing-title bugs. Pages silently inherit the brand name as title → garbage SERP listings.

**Fix:** Bare `{{ $title }}`. Let it throw.

## 3. `hreflang` without per-locale URLs

```html
<link rel="alternate" hreflang="en" href="https://yourdomain.com/">
<link rel="alternate" hreflang="cs" href="https://yourdomain.com/?locale=cs">
```

**Why bad:** Cookie/session-based locale means both URLs serve the same content to crawlers. Google penalizes hreflang implementation errors.

**Fix:** Drop the entire `<link rel="alternate" hreflang>` block until URL-prefixed routes exist.

## 4. Locale-aware canonical without URL-prefixed locales

```html
<link rel="canonical" href="{{ $locale === $defaultLocale ? $url : $url.'?locale='.$locale }}">
```

**Why bad:** Same lie as hreflang. `?locale=cs` isn't a real URL the server reads.

**Fix:** Bare `<link rel="canonical" href="{{ url()->current() }}">`.

## 5. WebP / AVIF `og:image` alternates

```html
<meta property="og:image" content="og/home.png">
<meta property="og:image:type" content="image/png">
<meta property="og:image" content="og/home.webp">
<meta property="og:image:type" content="image/webp">
<meta property="og:image" content="og/home.avif">
<meta property="og:image:type" content="image/avif">
```

**Why bad:** No major social scraper consumes WebP/AVIF. Tags + files are dead weight.

**Fix:** Single PNG. Delete `.webp` + `.avif` files from `public/og/`.

## 6. Locale-suffixed OG images with cookie-based locale

```blade
<x-slot:ogImage>{{ asset('og/home-'.app()->getLocale().'.png') }}</x-slot:ogImage>
```

**Why bad:** Social scrapers have no cookie → always render fallback locale → never fetch `-cs.png`. Bytes wasted on disk.

**Fix:** `og/<slug>.png` with no locale suffix.

## 7. `og:image:secure_url`

```html
<meta property="og:image:secure_url" content="https://yourdomain.com/og/home.png">
```

**Why bad:** Deprecated. Only useful when serving mixed-content. HTTPS-only sites don't need it.

**Fix:** Drop.

## 8. `og:see_also` cross-link map

```blade
@foreach (['https://yourdomain.com/docs', 'https://yourdomain.com/privacy'] as $url)
    <meta property="og:see_also" content="{{ $url }}">
@endforeach
```

**Why bad:** Niche field, few consumers, weak signal. Discovery happens via footer links + sitemap.

**Fix:** Drop.

## 9. `article:*` tags hardcoded layout-wide

```html
<meta property="article:published_time" content="2026-06-10">
<meta property="article:modified_time" content="2026-06-10">
<meta property="article:author" content="Martin Macháček">
<meta property="article:section" content="Legal">
```

**Why bad:** Same date on every page = lie. Article tags are low signal anyway. JSON-LD Article schema is the better channel.

**Fix:** Drop OG article:* tags. Put dates in JSON-LD `Article.datePublished` / `dateModified` if needed. Render visible "Last updated:" line in body for legal compliance.

## 10. `mask-icon` for legacy Safari pinned tabs

```html
<link rel="mask-icon" href="/favicon.svg" color="#0a0a0a">
```

**Why bad:** Legacy (pre Safari 12). Requires monochrome silhouette SVG (not regular favicon). Safari 12+ uses `rel="icon"` SVG directly.

**Fix:** Drop.

## 11. OpenSearch autodiscovery without site search

```html
<link rel="search" type="application/opensearchdescription+xml" title="..." href="/opensearch.xml">
```

**Why bad:** Useful only when site has `?q=` search the browser can register. Most SaaS sites don't.

**Fix:** Drop. Also delete `public/opensearch.xml`.

## 12. `rel="me"` to non-existent accounts

```html
<link rel="me" href="https://x.com/notdowndev">
```

**Why bad:** Account doesn't exist → Mastodon verification fails → IndieAuth chains break.

**Fix:** Only ship `rel="me"` for accounts that exist AND link back to your domain. Write a TODO plan file for accounts you'll create later.

## 13. `dns-prefetch` for `rel="me"` targets

```html
<link rel="dns-prefetch" href="https://github.com">
<link rel="dns-prefetch" href="https://x.com">
```

**Why bad:** No page-load fetch happens to these domains. DNS lookups wasted.

**Fix:** Drop. Keep `preconnect`/`dns-prefetch` only for hosts you actually fetch from (font CDN, image CDN, analytics).

## 14. `<meta http-equiv="content-language">`

```html
<meta http-equiv="content-language" content="en">
```

**Why bad:** Legacy, superseded by `<html lang>`.

**Fix:** Drop.

## 15. Pure white `theme-color` on warm-canvas site

```html
<meta name="theme-color" content="#ffffff">
```

**Why bad:** Doesn't match the actual `bg-canvas` color. Mobile browser chrome flashes white before page CSS loads → visible color mismatch with page bg.

**Fix:** Use exact hex of canvas color (e.g., `#f6f4ec` for warm off-white).

## 16. Dark `theme-color` variant with no dark mode in CSS

```html
<meta name="theme-color" content="#ffffff" media="(prefers-color-scheme: light)">
<meta name="theme-color" content="#0a0a0a" media="(prefers-color-scheme: dark)">
```

**Why bad:** If site CSS has no dark mode, the dark `theme-color` results in browsers showing dark chrome above LIGHT page content for users in dark mode → ugly.

**Fix:** Single `<meta name="theme-color" content="#brand-canvas">`. Set `color-scheme: light` (not `light dark`).

## 17. `@stack('head')` with no `@push` consumers

```html
@stack('head')
```

**Why bad:** Dead code if no pages `@push('head')` anything.

**Fix:** Grep for `@push('head')`. If zero matches, drop the `@stack`.

## 18. `<meta name="generator" content="Laravel">`

**Why bad:** Info leak. Tells attackers your stack. Zero SEO value.

**Fix:** Drop everywhere.

## 19. Behind-login layouts with full SEO meta

App / dashboard / guest auth layouts emitting OG tags, JSON-LD, rel=me, canonical, webmaster verification.

**Why bad:** These pages can't be crawled. Tags = bytes per request for zero signal. App users hit dozens of pages/session → real perf cost.

**Fix:** Strip behind-login layouts to UX-only meta. See `behind-login.md`.

## 20. Sitemap `image:title` = URL slug

```xml
<image:title>home · Notdown</image:title>
```

**Why bad:** "home" is an internal identifier, not a title. Google Image Search uses image:title verbatim.

**Fix:** Pull from translation keys:
```php
'title' => trans('pages.home.meta.title', [], 'en')
```

Result:
```xml
<image:title>Notdown — All-in-one monitoring for AI-driven development</image:title>
```

## 21. Per-locale sitemap entries with cookie-based locale

```xml
<url>
    <loc>https://yourdomain.com/</loc>
    <xhtml:link rel="alternate" hreflang="cs" href="https://yourdomain.com/?locale=cs"/>
</url>
<url>
    <loc>https://yourdomain.com/?locale=cs</loc>
    ...
</url>
```

**Why bad:** `?locale=cs` URL serves EN content to crawlers (no cookie). Sitemap is full of lies.

**Fix:** Single `<url>` per page. Drop `xhtml:link` blocks. Drop `xmlns:xhtml` namespace declaration.

## 22. Twitter card duplicating OG without `twitter:card` type

```html
<meta name="twitter:title" content="...">
<meta name="twitter:description" content="...">
<!-- but no <meta name="twitter:card"> -->
```

**Why bad:** Without `twitter:card`, X falls back to plain summary (small thumb). The image goes unused.

**Fix:** Always include `<meta name="twitter:card" content="summary_large_image">`.

## 23. Webmaster verification on behind-login pages

**Why bad:** Verification meta only needs to render on pages Google can crawl to confirm site ownership. Behind-login pages can't be crawled → emission is wasted.

**Fix:** Move webmaster verification block to marketing + public layouts only.

## 24. `Last update: 2026-06-10` hardcoded in `humans.txt`

**Why bad:** Rots immediately. Becomes a lie within weeks.

**Fix:** Either delete the line or regenerate humans.txt via `seo:` command using `date('Y-m-d')`. Manual updates inevitably stop.

## Cleanup checklist

When auditing a layout you didn't write, search for these patterns and decide intentionally on each:

```bash
grep -E '@php|hreflang|webp|avif|secure_url|see_also|article:|mask-icon|opensearchdescription|content-language|@stack|generator|x.com/(notdowndev|<dead-handle>)' resources/views/components/layouts/*.blade.php
```

Most of those hits should be deletions.
