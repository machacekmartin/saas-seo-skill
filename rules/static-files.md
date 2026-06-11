# Static SEO Files

Files that live under `public/` and are served as-is. Critical infrastructure that's easy to forget and easy to let rot.

## robots.txt

Tells crawlers what they can/can't fetch, where the sitemap is, and (in the LLM era) explicitly allowlists AI crawlers.

```
User-agent: *
Allow: /

# AI crawlers — explicit allow
User-agent: GPTBot
Allow: /

User-agent: ClaudeBot
Allow: /

User-agent: PerplexityBot
Allow: /

# Regional crawlers (drop if not your audience)
User-agent: SeznamBot
Allow: /

# Sitemaps
Sitemap: https://yourdomain.com/sitemap.xml
Sitemap: https://yourdomain.com/sitemap-index.xml
```

Why explicit AI allowlist: most crawlers default to allowed when no rule matches, but explicit declaration shows intent and survives future `User-agent: *` Disallow blocks.

## sitemap.xml + sitemap-index.xml

Auto-generated from an Artisan command, NOT hand-written. The command reads file mtimes for `<lastmod>` and pulls real titles from translation files.

### Pattern

```php
// app/Console/Commands/GenerateSitemapCommand.php
class GenerateSitemapCommand extends Command
{
    protected $signature = 'seo:generate-sitemap';

    public function handle(): int
    {
        $base = 'https://yourdomain.com';
        $viewsRoot = resource_path('views/pages');

        $mtime = fn (string $view) => file_exists("$viewsRoot/$view")
            ? filemtime("$viewsRoot/$view") : time();

        $pages = [
            ['loc' => "$base/", 'view' => 'home.blade.php', 'slug' => 'home',
             'title' => trans('pages.home.meta.title', [], 'en')],
            // ... one entry per indexable page
        ];

        $xml = '<?xml version="1.0" encoding="UTF-8"?>'."\n";
        $xml .= '<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9"'."\n";
        $xml .= '        xmlns:image="http://www.google.com/schemas/sitemaps-image/1.1">'."\n";

        foreach ($pages as $page) {
            $xml .= "  <url>\n";
            $xml .= "    <loc>{$page['loc']}</loc>\n";
            $xml .= "    <lastmod>".date('Y-m-d', $mtime($page['view']))."</lastmod>\n";

            $img = "og/{$page['slug']}.png";
            if (file_exists(public_path($img))) {
                $xml .= "    <image:image>\n";
                $xml .= "      <image:loc>$base/$img</image:loc>\n";
                $xml .= "      <image:title>".htmlspecialchars($page['title'], ENT_XML1)."</image:title>\n";
                $xml .= "    </image:image>\n";
            }
            $xml .= "  </url>\n";
        }

        $xml .= "</urlset>\n";
        file_put_contents(public_path('sitemap.xml'), $xml);

        // sitemap-index.xml points at sitemap.xml
        // ...

        return self::SUCCESS;
    }
}
```

### Key choices

- **No `xmlns:xhtml` namespace, no `<xhtml:link rel="alternate" hreflang>` blocks** — unless URL-prefixed locales exist. See `locale.md`.
- **Real `<image:title>`** — pull from translation files (`trans('pages.X.meta.title', [], 'en')`), NOT from slug. Sitemap-image is used in Google Image Search.
- **One `<url>` per page**, not per locale × page combo. Default locale only.
- **`<lastmod>` from view mtime** — auto-updates whenever you edit a page. Cron / CI runs `php artisan seo:generate-sitemap` on deploy.

### sitemap-index.xml

Even with a single sitemap, ship the index. Google prefers index pointing at sitemap.xml; future-proofs for when you split.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<sitemapindex xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <sitemap>
    <loc>https://yourdomain.com/sitemap.xml</loc>
    <lastmod>2026-06-11T00:00:00Z</lastmod>
  </sitemap>
</sitemapindex>
```

Both sitemap and index referenced in robots.txt.

## llms.txt (per llmstxt.org spec)

LLM-era equivalent of robots.txt: tells AI crawlers what your site is about and which pages are canonical references.

```
# Notdown

> All-in-one monitoring for AI-driven development.

## Docs

- [MCP server reference](https://yourdomain.com/docs/mcp): Connect Claude, Cursor, ChatGPT to your monitors.

## Legal

- [Privacy Policy](https://yourdomain.com/privacy)
- [Terms of Service](https://yourdomain.com/terms)
- [Cookie Policy](https://yourdomain.com/cookies)
```

Format spec: https://llmstxt.org. Markdown, one-line description after `>` quote, sections of `- [Title](url): one-line description.`

## llms-full.txt

Longer form — drop the entire product reference here. AI agents can fetch one file and have ground truth about your product.

```
# Notdown — Full Product Reference

## What Notdown does
<concise product description>

## Frequently asked questions
<inlined FAQs>

## Programmatic surfaces
- REST API: <details>
- MCP server: <details>
- CLI: <details>

## URLs
<link list>
```

Strategy: write once, regenerate when product changes meaningfully. Validate via tests that key strings appear (`'MCP server'`, `'REST API'`, etc.).

## security.txt + .well-known/security.txt (RFC 9116)

Both paths required. Tells security researchers how to report vulnerabilities.

```
Contact: mailto:security@yourdomain.com
Expires: 2027-12-31T23:59:59Z
Encryption: https://yourdomain.com/pgp.asc
Acknowledgments: https://yourdomain.com/security/thanks
Policy: https://yourdomain.com/security/policy
Canonical: https://yourdomain.com/.well-known/security.txt
Preferred-Languages: en, cs
```

`Expires:` must be a future date; rotate before it expires.

## humans.txt

Team credits + tech stack. Mostly cultural.

```
/* TEAM */

Founder & Engineer: Your Name
Location: Czechia
Site: https://yourdomain.com
GitHub: https://github.com/<real-handle>

/* SITE */

Last update: 2026/06/10
Language: English
Components: Laravel 13, Livewire 4, Tailwind 4.
```

**Critical:** only list accounts that exist. No "Mastodon / X: @notdowndev" if the account is vapor.

## manifest.webmanifest

PWA install + iOS/Android home screen behavior.

```json
{
  "name": "Notdown",
  "short_name": "Notdown",
  "description": "All-in-one monitoring for AI-driven development.",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#f6f4ec",
  "theme_color": "#f6f4ec",
  "icons": [
    { "src": "/icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icons/icon-512.png", "sizes": "512x512", "type": "image/png" },
    { "src": "/icons/icon-maskable-512.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ]
}
```

Icons: at minimum 192 + 512 + maskable-512. Maskable means inset padding so Android can crop it into any shape (circle, squircle).

Validate via test:
```php
$manifest = json_decode(file_get_contents(public_path('manifest.webmanifest')), true);
expect($manifest)
    ->toHaveKey('name', 'Notdown')
    ->toHaveKey('display', 'standalone')
    ->toHaveKey('icons');
expect($manifest['icons'])->toHaveCount(3);
```

## favicon.svg with dark-mode media query

SVG favicons can embed `<style>` with `prefers-color-scheme` so the icon swaps colors based on user theme:

```xml
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 32 32">
  <style>
    .fg { fill: #0a0a0a; }
    @media (prefers-color-scheme: dark) { .fg { fill: #f6f4ec; } }
  </style>
  <path class="fg" d="..."/>
</svg>
```

Result: light theme browsers show dark icon, dark theme browsers show light icon. Works in tab + bookmarks.

## What to delete from a typical scaffolded `public/`

- `.htaccess` — irrelevant on Laravel Cloud / nginx. Delete unless on Apache.
- `opensearch.xml` — drop unless site has real `?q=` search.
- `og/*-cs.png`, `og/*-en.png` (locale-suffixed) — simplify to `og/<slug>.png` if single-locale.
- `og/*.webp`, `og/*.avif` — no social scraper consumes these. PNG only.

## Regeneration cadence

- Sitemap + sitemap-index: regenerated by `php artisan seo:generate-sitemap` on every deploy (CI step).
- llms.txt + llms-full.txt: regenerated manually when product copy changes.
- security.txt: rotate `Expires:` annually.
- humans.txt: update on team changes.
- manifest.webmanifest: rarely changes.
- robots.txt: rarely changes.
