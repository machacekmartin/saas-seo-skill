# Locale-Aware Indexing

The most common SEO cargo cult: shipping `hreflang` and `og:locale:alternate` tags on a site where every locale lives at the same URL.

## The decision tree

**Question:** Does each locale have a distinct URL Google can crawl?

- **Yes** — `/cs/privacy` vs `/en/privacy` (URL prefix), or `cs.domain.com` (subdomain), or `domain.cz` (separate domain). → Ship full hreflang + per-locale canonicals + per-locale OG images.
- **No** — Same URL, locale set by session/cookie/Accept-Language. → Drop ALL locale signals. Accept single-locale indexing (whichever is your `fallback_locale`).

## Why "no" is so common — and so often mis-handled

Most Laravel apps use middleware like:
```php
$locale = $request->session()->get('locale')
    ?? $request->cookie('locale')
    ?? config('app.fallback_locale');
App::setLocale($locale);
```

Every page is served on the same URL regardless of locale. Locale switching is a `/locale/{cs}` route that sets a cookie and `redirect()->back()`.

**Googlebot has no cookie.** Every Googlebot request gets the fallback locale. The Czech version of `/privacy` is invisible to Google — the URL `/privacy` always returns EN content to crawlers.

Shipping `hreflang="cs" href="https://yourdomain.com/?locale=cs"` is a lie:
1. Google follows it. Hits `/?locale=cs`. Server reads session (empty) → serves EN content.
2. Google compares to canonical. Sees they differ.
3. Penalizes site for hreflang implementation errors.
4. Czech version still not indexed.

Net result: actively worse than not shipping hreflang at all.

## What to drop (no URL-prefix locales)

In layout:
- `<link rel="alternate" hreflang="en">`, `hreflang="cs"`, `hreflang="x-default"` — entire block
- `<meta property="og:locale:alternate" content="cs_CZ">` (or any non-default locale)
- Locale-aware canonical (`<link rel="canonical">` should be bare `url()->current()`)
- Locale-aware `og:url`

In sitemap.xml:
- Per-locale `<url>` entries with `?locale=cs`
- `<xhtml:link rel="alternate" hreflang>` inside `<url>` blocks
- `xmlns:xhtml` namespace declaration

In OG image references:
- `og/<page>-{locale}.png` → simplify to `og/<page>.png`
- Delete `-cs.png` files (Googlebot never fetches them)

In static files:
- `humans.txt` / `llms.txt` references to `?locale=cs` URLs

Hardcode `og:locale` to your fallback (e.g., `en_US`):
```html
<meta property="og:locale" content="en_US">
```

## What to keep (with URL-prefix locales)

If you DO have proper per-locale URLs:

```html
<link rel="alternate" hreflang="en" href="https://yourdomain.com/en/privacy">
<link rel="alternate" hreflang="cs" href="https://yourdomain.com/cs/privacy">
<link rel="alternate" hreflang="x-default" href="https://yourdomain.com/en/privacy">

<link rel="canonical" href="https://yourdomain.com/en/privacy">

<meta property="og:locale" content="en_US">
<meta property="og:locale:alternate" content="cs_CZ">
```

Sitemap has separate `<url>` entries per locale with reciprocal hreflang links inside each.

Per-locale OG images at `og/<page>-{locale}.png` — and they're actually fetchable because the URL determines locale.

## How to inform Google about a second locale you don't expose

Short answer: you can't, with cookie-based locale. The architecture is the constraint.

Options to enable proper multilingual SEO later:
1. **URL prefix routes** — most common. Refactor middleware to read locale from URL segment, redirect `/privacy` → `/en/privacy` or `/cs/privacy` based on `Accept-Language` for new visitors.
2. **Subdomain** — `cs.domain.com`, `en.domain.com`. More infra (TLS, DNS).
3. **Separate domain** — `.cz` vs `.com`. Heaviest.

Until one of those exists, the second locale serves only existing users (who have the cookie). It's a UX feature, not an SEO feature.

## What NOT to try

- `Vary: Accept-Language` header — Google generally ignores it for indexing different versions.
- Content negotiation via `Accept-Language` — Googlebot doesn't send it consistently.
- Server-side IP geo-detection — Google explicitly recommends against this.
- JavaScript-based locale switching — invisible to crawlers.

## Tagging this in the codebase

If you accepted single-locale indexing now and plan to URL-prefix later, write a plan file:

```
docs/superpowers/plans/YYYY-MM-DD-locale-url-prefix.md
```

Documenting: current state (cookie-based, EN-only indexing), trigger (when CS organic traffic matters), what changes (middleware, routes, layouts, sitemap, OG images).
