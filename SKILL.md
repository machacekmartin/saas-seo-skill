---
name: seo-foundation
description: "Apply this skill whenever wiring SEO into a web app — head meta tags, Open Graph, Twitter cards, JSON-LD schema.org, hreflang, canonical, robots/sitemap/llms.txt, OG image strategy, favicon/manifest/PWA meta, AI crawler optimization, theme-color, or webmaster verification. Trigger when designing layout files, refactoring head blocks, adding social unfurls, building a public marketing site that needs to rank, debugging why a page isn't indexed, or auditing an existing layout for SEO cargo-cult. Also use for behind-login pages where SEO must be stripped down to UX meta only, and for deciding which static files (robots.txt, sitemap.xml, llms.txt, security.txt, humans.txt, manifest.webmanifest) the site needs."
license: MIT
metadata:
  author: machacek
---

# SEO Foundation

Battle-tested SEO setup for a SaaS website. Distilled from a full audit that ripped out cargo-cult meta tags and rebuilt the layout / sitemap / static files around what crawlers and unfurl scrapers actually consume in 2026.

## Core Principles

**Cargo cult is the enemy.** Every meta tag has a cost (template noise, ongoing maintenance, lies to crawlers) and a benefit (signal to a real consumer). Default to drop; require justification to ship. Most "comprehensive SEO checklist" posts list tags no major consumer reads.

**Match architecture to indexability.** Pages split cleanly into three buckets:
1. **Public + indexable** (home, pricing, docs, legal) — full SEO meta + JSON-LD + OG/Twitter.
2. **Public + noindex** (status pages controlled by `noindex` prop, subscribe-verify flows) — same meta tags, robots flips to noindex.
3. **Behind login** (dashboards, settings, auth screens) — strip ALL SEO. Keep only UX meta (favicon, theme-color, viewport, csrf, manifest). No OG, no canonical, no JSON-LD.

**Don't lie to Google.** If your locales aren't on distinct URLs, don't fake hreflang. If your X handle doesn't exist, don't ship `rel="me"` to it. Broken signals damage trust + waste crawl budget.

**Inline > computed variables in templates.** Heavy `@php` blocks computing `$titleStr`, `$resolvedCanonical`, etc. are an anti-pattern. Use direct expressions at usage sites; force pages to fill required slots (`$title`, `$description`, `$ogImage`) so missing values throw at render rather than silently emit empty meta.

**Required slots > fallbacks.** Layouts that compute fallbacks for missing title/description hide bugs. Make slots required → Blade throws `Undefined variable: $title` if a page forgets one. Discovery at render is cheaper than discovery in production.

**Real titles, not slugs.** `<image:title>home · Notdown</image:title>` in sitemap = wrong. Wire actual page titles (from translation files) into the sitemap generator.

**Brand-first em-dash for titles.** `"Notdown — Page topic"` everywhere. Brand recall + visual consistency + no mixed `· vs —` chaos.

## Quick Reference

### 1. Layout architecture → `rules/architecture.md`

Three-tier layout pattern: marketing (indexable), public (mixed via `noindex` prop), app + guest (behind-login, stripped). What each emits, what each drops.

### 2. Head meta tags — keep/drop catalog → `rules/meta-tags.md`

Every common head tag with verdict + reasoning. charset/viewport/csrf (keep), generator (drop, info leak), content-language (drop, legacy), mask-icon (drop, legacy), opensearch (drop unless real site search), theme-color (keep matched to brand canvas — no `prefers-color-scheme: dark` variant unless site has dark mode), favicon set (svg + ico + apple-touch-icon + manifest).

### 3. Open Graph + Twitter cards → `rules/open-graph.md`

Single PNG 1200×630, no WebP/AVIF alternates (scrapers ignore them), no `og:image:secure_url` (deprecated), `og:locale` hardcoded, no `og:locale:alternate` unless URL-prefixed locales exist. Twitter tags duplicate OG — keep only if your X handle is real.

### 4. JSON-LD schema.org → `rules/json-ld.md`

High-ROI structured data: Organization (with ContactPoint, foundingLocation, knowsLanguage), WebSite, SoftwareApplication (with Offer + UnitPriceSpecification for SaaS), Person (founder, E-E-A-T), FAQPage (with SpeakableSpecification for voice), DefinedTermSet (topical authority), BreadcrumbList, Article/WebPage/TechArticle/HowTo. One shared `<x-seo.json-ld>` component, page-driven schema selection.

### 5. Locale + hreflang → `rules/locale.md`

If locale is session/cookie-based (same URL, different language), Google can't index both — period. Don't ship `hreflang` or `og:locale:alternate` or per-locale `?locale=cs` canonical URLs — they're lies. Decision tree: accept single-locale indexing now, or refactor to URL-prefixed routes (`/cs/...`).

### 6. Behind-login pages → `rules/behind-login.md`

Strip every SEO meta from app + guest layouts. Keep only: charset, viewport, csrf-token, title, robots noindex, favicon set, manifest, theme-color, color-scheme, apple-mobile-web-app-* tags, format-detection, preconnect for fonts. Drop description, og:*, twitter:*, canonical, rel=me, rel=author, application-name (debatable), generator, webmaster verification, JSON-LD.

### 7. Static files → `rules/static-files.md`

robots.txt (Sitemap line + AI crawler allowlist), sitemap.xml + sitemap-index.xml (generated from artisan command, real titles), llms.txt + llms-full.txt (per llmstxt.org), security.txt + .well-known/security.txt (RFC 9116), humans.txt, manifest.webmanifest, favicon.svg with dark-mode media query embedded inside the SVG.

### 8. OG image strategy → `rules/og-images.md`

1200×630 PNG, one variant per page slug (`og/home.png`, `og/privacy.png`, etc.). No locale suffix (scrapers have no cookie → can't fetch -cs variant anyway). No webp/avif (no real consumer). Required per-page slot — every page sets its own.

### 9. Titles + descriptions → `rules/titles.md`

Consistent pattern: `"Brand — Page topic"`. All from translation files. Sitemap pulls same translation key so SERP image labels match `<title>`. Descriptions are page-specific; no app-wide fallback in layout.

### 10. AI crawler optimization → `rules/ai-crawlers.md`

LLM-era SEO: llms.txt + llms-full.txt (per llmstxt.org spec), JSON-LD DefinedTermSet for topical authority, FAQPage with SpeakableSpecification, robots.txt explicit allow for GPTBot / ClaudeBot / PerplexityBot / regional crawlers (SeznamBot for CZ/SK). data-speakable attributes on hero + key content.

### 11. Anti-patterns → `rules/anti-patterns.md`

Tags + patterns to actively REMOVE on sight. The "comprehensive SEO setup" you inherited likely contains all of these.

## Workflow When Auditing a Layout

1. Read each `<meta>` / `<link>` / `<script>` in `<head>` one by one.
2. For each, ask: who consumes this? what do they do with it? does the consumer exist for our site (e.g., is the X account real)?
3. If no real consumer or the signal is broken, drop it.
4. After dropping, verify nothing else (tests, JS, other templates) still references it.
5. Required slots check: every layout that uses `{{ $title }}` etc. should throw without fallback. Audit pages to make sure each fills required slots.
6. Match sitemap + llms.txt + humans.txt + security.txt + robots.txt to the new reality (no orphan references to dropped accounts, dropped locales, dropped image variants).

## Workflow When Bootstrapping SEO in a New Project

1. Decide indexability per layout (public + indexable, public + noindex, behind-login).
2. Build layouts with required slots — `$title`, `$description`, `$ogImage` for indexable, just `$title` for noindex.
3. Drop fallbacks in layout — force pages to provide.
4. Ship single PNG OG images per page slug at 1200×630.
5. Ship robots.txt + sitemap-index.xml + sitemap.xml + llms.txt + security.txt + humans.txt + manifest.webmanifest + favicon set.
6. JSON-LD: Organization + WebSite + (SoftwareApplication if SaaS) at minimum. Add Person (founder), FAQPage, DefinedTermSet as content matures.
7. Webmaster verification config-driven via env vars (`config/seo.php` with `env('SEO_VERIFY_GOOGLE')` etc.). @if-guarded in layout.
8. Locale decision up front: URL-prefixed routes (proper hreflang) or accept single-locale indexing (drop hreflang, drop locale-suffixed OG images).
