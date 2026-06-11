# AI Crawler Optimization (LLM-Era SEO)

LLM crawlers (GPTBot, ClaudeBot, PerplexityBot, etc.) read pages to train + answer user queries. They reward different signals than traditional SEO: structured definitions, FAQ pairs, single-fetch product references.

## The four pillars

1. **`robots.txt` explicit allow** for AI crawlers
2. **`llms.txt` + `llms-full.txt`** per llmstxt.org spec
3. **JSON-LD `DefinedTermSet`** for topical authority
4. **JSON-LD `FAQPage` + speakable markers** for retrieval

## robots.txt — explicit AI allowlist

Default crawler rules allow most user-agents, but being explicit signals intent and survives future `Disallow` changes:

```
User-agent: GPTBot
Allow: /

User-agent: ClaudeBot
Allow: /

User-agent: PerplexityBot
Allow: /

User-agent: anthropic-ai
Allow: /

User-agent: cohere-ai
Allow: /

User-agent: Google-Extended
Allow: /

User-agent: Bytespider
Disallow: /
```

(Disallow Bytespider if you don't want ByteDance/TikTok training on your content. Many sites do.)

Test that robots.txt advertises the bots you care about:
```php
expect($body)
    ->toContain('GPTBot')
    ->toContain('ClaudeBot')
    ->toContain('PerplexityBot');
```

## llms.txt

Per https://llmstxt.org spec. One markdown file at `/llms.txt`. AI agents fetch this to understand what your site is about.

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

Format rules:
- H1 = product name only
- `>` quote = one-line product description (same as og:description)
- H2 sections group canonical pages
- Each entry: `- [Title](url): one-line description.`

Keep it under 100 lines. This is the index, not the encyclopedia.

## llms-full.txt

Long-form companion. The full product reference dumped into one file an agent can fetch and have ground truth about your product.

Structure suggestion:
```
# Notdown — Full Product Reference

## What Notdown does
<2-paragraph description>

## What problems it solves
<bullet list>

## How it works
<sections per monitor type, API, MCP, CLI>

## Frequently asked questions
<full FAQ Q&A inline>

## Programmatic surfaces
### REST API
<endpoints + auth>

### MCP server
<server URL + tool list + capabilities>

### CLI
<install + commands>

## URLs
- Home: https://yourdomain.com
- Docs: https://yourdomain.com/docs/mcp
- ...
```

Why this works for LLM agents:
- One fetch = full context. No multi-hop crawling needed.
- Markdown structure = clean tokenization.
- Q&A pairs in body = high retrieval signal.
- Inline URL list = canonical references.

Validate via test that key sections survive edits:
```php
expect($body)
    ->toStartWith('# Notdown — Full Product Reference')
    ->toContain('## What Notdown does')
    ->toContain('## Frequently asked questions')
    ->toContain('## Programmatic surfaces')
    ->toContain('REST API')
    ->toContain('MCP server');
```

## JSON-LD `DefinedTermSet` (the secret weapon)

Defines your domain's vocabulary. Massively helpful for LLM retrieval — Claude/GPT can match user queries to your defined terms verbatim.

```json
{
  "@context": "https://schema.org",
  "@type": "DefinedTermSet",
  "@id": "https://yourdomain.com/#glossary",
  "name": "Monitoring glossary",
  "hasDefinedTerm": [
    {
      "@type": "DefinedTerm",
      "name": "HTTP uptime monitoring",
      "description": "Periodic HTTP requests against a URL on a fixed schedule. The check fails when the response status code is outside the configured success range, when content assertions do not match, or when the request times out. A failed check opens an incident; consecutive failures across confirmation locations are required before alerts are sent.",
      "termCode": "http_uptime"
    },
    {
      "@type": "DefinedTerm",
      "name": "Cron heartbeat monitoring",
      "description": "..."
    }
  ]
}
```

Rules:
- 5–10 terms per page (not more — dilutes signal)
- 2–4 sentence definitions. Concrete, quotable, unambiguous.
- Use the term naturally in body text as well (not just in JSON-LD)
- `termCode` is a stable machine identifier — useful for cross-referencing

## JSON-LD `FAQPage` with `speakable`

```json
{
  "@type": "FAQPage",
  "@id": "https://yourdomain.com/#faq",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "What is HTTP uptime monitoring?",
      "acceptedAnswer": { "@type": "Answer", "text": "..." }
    }
  ],
  "speakable": {
    "@type": "SpeakableSpecification",
    "cssSelector": ["[data-speakable]", "h1", "h2"]
  }
}
```

Why FAQPage for LLM SEO:
- Direct Q&A retrieval signal
- LLMs love structured pairs
- `speakable` opens voice assistant integration (Siri Reader, Google Assistant)

Pair with visible body HTML using microdata:
```html
<div itemscope itemtype="https://schema.org/FAQPage">
    <div itemprop="mainEntity" itemscope itemtype="https://schema.org/Question">
        <h3 itemprop="name" data-speakable>What is HTTP uptime monitoring?</h3>
        <div itemprop="acceptedAnswer" itemscope itemtype="https://schema.org/Answer">
            <p itemprop="text" data-speakable>...</p>
        </div>
    </div>
</div>
```

Now Q&A is BOTH in JSON-LD (for crawler parse) AND in visible HTML (for rendering + microdata fallback). Belt and suspenders.

## `data-speakable` attributes

Mark hero + key content with `data-speakable` so voice assistants know what to read aloud:

```html
<h1 data-speakable class="text-5xl">Notdown — All-in-one monitoring</h1>
<p data-speakable class="text-xl">Track uptime, crons, SSL, and domain expiry...</p>
```

Cheap signal. Test that key pages have plenty of speakable markers:
```php
$body = $r->getContent();
expect(substr_count($body, 'data-speakable'))->toBeGreaterThanOrEqual(30);
```

## Noscript SEO fallback

Some LLM crawlers and traditional crawlers don't execute JS. Marketing layout should ship a `<noscript>` block with the page essence:

```html
<noscript>
    <div role="region" aria-label="Notdown — text summary">
        <h1>Notdown — All-in-one monitoring for AI-driven development.</h1>
        <p>{{ __('pages.home.meta.description') }}</p>
        <ul>
            <li><a href="{{ route('docs.mcp') }}">MCP server reference</a></li>
            <li><a href="{{ route('legal.privacy') }}">Privacy Policy</a></li>
            <li><a href="/llms.txt">llms.txt</a> · <a href="/sitemap.xml">sitemap.xml</a></li>
        </ul>
    </div>
</noscript>
```

Visible only when JS disabled. Contains: H1, page description, key links to legal + docs + machine-readable files.

## Test signals worth keeping in CI

- robots.txt mentions GPTBot/ClaudeBot/PerplexityBot
- llms.txt starts with `# <product>` and contains key URLs
- llms-full.txt has required sections (What X does, FAQ, surfaces, URLs)
- FAQPage JSON-LD has expected question count
- DefinedTermSet has expected term names
- `data-speakable` count above threshold

These prevent silent regressions when someone refactors the home page.
