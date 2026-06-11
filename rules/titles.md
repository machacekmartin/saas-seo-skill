# Titles + Descriptions

The two highest-impact pieces of text on every page. SERP click-through, social unfurl impression, browser tab UX.

## Title pattern: `Brand — Page topic`

Single consistent pattern across every indexable page. Em-dash separator (` — `), brand FIRST, page topic second.

```
Notdown — All-in-one monitoring for AI-driven development
Notdown — MCP server
Notdown — Privacy Policy
Notdown — Terms of Service
Notdown — Cookie Policy
```

### Why brand-first em-dash

- **Brand recall.** First chars are most-read. SERP truncates at ~60 chars. Brand survives truncation.
- **Visual consistency.** No mixed `· vs — vs |` chaos. Same separator everywhere.
- **Multilingual coherence.** Em-dash reads the same in EN/CS/DE/etc. Period/colon/middot have language-specific conventions.
- **Sitemap + browser tab consistency.** What appears in `<title>` should match what appears in `<image:title>` in sitemap.

### Alternatives (pick ONE)

If you absolutely need page-first (e.g., topic discovery matters more than brand recall — wiki sites, documentation):
```
Privacy Policy · Notdown
```

Stick with one across every page. Mixing kills the consistency win.

## Title source: translation files

Never hardcode titles in Blade. Always reference translation keys:

```blade
<x-slot:title>{{ __('pages.legal.privacy.meta.title') }}</x-slot:title>
```

`lang/en.json`:
```json
{
  "pages.home.meta.title": "Notdown — All-in-one monitoring for AI-driven development",
  "pages.legal.privacy.meta.title": "Notdown — Privacy Policy",
  "pages.legal.terms.meta.title": "Notdown — Terms of Service",
  "pages.legal.cookies.meta.title": "Notdown — Cookie Policy",
  "pages.docs.mcp.meta.title": "Notdown — MCP server"
}
```

Why translation files:
- Source of truth — sitemap generator can pull from same key via `trans('pages.X.meta.title', [], 'en')`.
- Pattern visible at a glance when reviewing `en.json` — diverging titles jump out.
- i18n-ready for the day you ship URL-prefixed locales.

## Title sizing

- Target 50–60 characters total. Google truncates ~580px which is roughly 60 chars at the SERP font.
- `Notdown — ` is 10 chars → page topic gets 40–50.
- If page topic doesn't fit, shorten the topic, NOT the brand. Brand recall wins.

Bad: `Notdown — MCP server · Connect Claude, Cursor, ChatGPT, and other AI agents to your monitors` (way too long)
Good: `Notdown — MCP server` (clean, brand visible)

## Description pattern

`<meta name="description">` for SERP snippet + OG/Twitter body text.

- 130–160 characters
- Action-oriented or value-oriented opening
- Include 1–2 keywords naturally (not stuffed)
- No emoji (some SERPs strip them, lower professionalism)
- Single sentence preferred

```json
{
  "pages.home.meta.description": "All-in-one monitoring platform for developers. Track HTTP uptime, cron job heartbeats, SSL certificate expiry, and domain WHOIS — with public status pages, smart alerts, a REST API, a CLI, and an MCP server for AI agents."
}
```

## Required slots, no fallbacks

Layout uses `{{ $title }}` and `{{ $description }}` with no `?? 'default'`. Every page must set both via:
- `<x-slot:title>{{ __('key') }}</x-slot:title>` and `<x-slot:description>{{ __('key') }}</x-slot:description>`
- OR Livewire `#[Title('Page name')]` for title only (handy when no description needed because the page is noindex)

Behind-login pages skip description (drop the slot from layout and pages).

## What NOT to do

### Don't compute the title in layout

Anti-pattern:
```php
// in layout @php block
$fullTitle = $titleStr && ! str_contains($titleStr, $appName)
    ? $titleStr.' · '.$appName
    : ($titleStr ?: $appName);
```

This guesses what the page meant. Pages should pass the FINAL title including brand. Layout just echoes `{{ $title }}`. Discipline at the page level, not magic in the layout.

### Don't append site name in layout

Anti-pattern: `<title>{{ $title }} · {{ config('app.name') }}</title>`

Site name belongs in the translation string. Otherwise:
- Pages can't override (some legal docs read better without brand suffix).
- Sitemap titles must duplicate the logic when pulling from translations.
- Multilingual translations might want different brand placement (CZ: `Notdown — Foo` vs SK: `Foo · Notdown`).

Easier: each translation key contains the full final title.

### Don't fall back to `__('pages.home.meta.description')` for other pages

If `/privacy` doesn't have its own description, you get "All-in-one monitoring platform..." on the SERP snippet for Privacy Policy. Confusing and useless.

Require pages to provide their own.

### Don't translate technical product names

`MCP server` stays `MCP server` in Czech, not `MCP-ový server` or `Server MCP`. Technical proper nouns don't translate.

## Tab UX

Browser tabs typically show 15–25 chars. Brand-first ensures the user can find your tab in a sea of others by looking for "Notdown" first.

Bad: 7 open tabs all starting with `Pric...`, `Priv...`, `Term...` — user can't tell apart.
Good: 7 open tabs all starting with `Notdown — ...` — user scans by what comes AFTER the brand.

This is why brand-first wins in tab-heavy SaaS UX, not just SERP.

## Verification

Pest test asserting current title appears:
```php
it('renders home title', function () {
    $r = $this->get('/');
    $r->assertSee('<title>Notdown — All-in-one monitoring for AI-driven development</title>', false);
});
```

When you change the translation, this test breaks → forces conscious decision.
