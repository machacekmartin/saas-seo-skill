# Behind-Login Pages

The single biggest SEO cleanup win in most apps: stripping SEO meta from layouts that serve pages behind authentication.

## Why behind-login pages need no SEO

- **Can't be crawled.** Googlebot has no session/cookie. Hitting `/dashboard` returns the login form, not the dashboard. Whatever meta the dashboard layout emits, Google never sees on the dashboard URL.
- **Can't be indexed.** Even if URL leaks (referer, screenshot, paste in chat), `noindex,nofollow` keeps it out of SERP.
- **Unfurls are useless.** If someone shares `/dashboard` in Slack, the recipient hits a login wall. A pretty OG image just lies about the content.
- **Recipient already authenticated → uses their own dashboard,** not yours.
- **More bytes per page load.** App users hit dozens of pages per session. Trimming 30+ meta tags = real perf win.

## The minimal behind-login `<head>`

```html
<!DOCTYPE html>
<html lang="...">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
    <meta name="csrf-token" content="{{ csrf_token() }}">

    <title>{{ $title }}</title>
    <meta name="robots" content="noindex,nofollow">

    <meta name="theme-color" content="#<brand-canvas>">
    <meta name="color-scheme" content="light">
    <meta name="apple-mobile-web-app-title" content="{{ config('app.name') }}">
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-status-bar-style" content="default">
    <meta name="format-detection" content="telephone=no">
    <meta name="mobile-web-app-capable" content="yes">

    <link rel="icon" href="/favicon.svg" type="image/svg+xml">
    <link rel="icon" href="/favicon.ico" sizes="32x32">
    <link rel="apple-touch-icon" href="/apple-touch-icon.png">
    <link rel="manifest" href="/manifest.webmanifest">

    <link rel="preconnect" href="https://fonts.bunny.net" crossorigin>
    <link rel="dns-prefetch" href="https://fonts.bunny.net">

    @vite([...])
    @livewireStyles
</head>
```

That's the full SEO-free layout. ~20 lines of head meta instead of ~80.

## What to KEEP

These are UX, not SEO:

- `<title>` — browser tab + bookmarks
- `noindex,nofollow` robots — defense if URL leaks
- Favicon set (svg/ico/apple-touch) — tab + bookmark + iOS home screen
- Manifest — PWA install
- Theme-color + color-scheme — mobile browser chrome
- Apple/mobile webapp meta — standalone mode if user pins to home screen
- format-detection — prevents iOS auto-linking app text (numeric monitor names, version strings, etc.) → `tel:` URIs
- Preconnect to fonts CDN — perf

## What to DROP

- `<meta name="description">` — no SERP snippet to populate
- All `og:*` — useless unfurl
- All `twitter:*` — same
- `<link rel="canonical">` — nothing canonical about a private page
- `<link rel="me">` — identity signal for crawlers
- `<link rel="author">` — same
- `<meta name="application-name">` — debatable; tiny value
- `<meta name="generator">` — info leak, drop everywhere
- Webmaster verification — only relevant where you'd register Search Console (home + public)
- JSON-LD — schema.org is for crawlers
- `googlebot` meta — no Googlebot ever reaches these pages
- DNS prefetch for `rel=me` targets — no fetch happens

## Pages stripping

For pages using app/guest layouts, remove all `<x-slot:description>` and `<x-slot:ogImage>` blocks. Pages keep `#[Title('Page name')]` Livewire attribute (still fills layout `$title` slot) OR `<x-slot:title>` block.

Strip script:
```bash
grep -rl "Layout('components.layouts.app')\|Layout('components.layouts.guest')" resources/views/pages/ | while read -r f; do
  awk '
    /<x-slot:description>/ { next }
    /<x-slot:ogImage>/ { next }
    { print }
  ' "$f" > "$f.tmp" && mv "$f.tmp" "$f"
done
```

## The verification leak question

Webmaster verification (`google-site-verification` etc.) on behind-login pages is technically harmless — env-gated, only renders if env is set. But conceptually wrong: Google only needs to see the verification meta on a page it CAN crawl. Drop from app/guest, keep on marketing/public.

## Edge cases

- **Status pages (`/status/{slug}`)** — these are PUBLIC but per-user. Use the `public` layout with full meta. They unfurl meaningfully ("Acme · Status — All systems operational").
- **Email verification page** — uses `app` layout (authenticated but pre-verified user). Behind-login rules apply.
- **Subscribe-verify / unsubscribe (status page subscriber flows)** — public but transactional. Use `public` layout with `:noindex="true"` so they don't pollute SERP.
- **404 page** — public. Full meta but `:noindex="true"`. Even error pages get the OG default image so a shared `404` URL doesn't unfurl as a broken card.
