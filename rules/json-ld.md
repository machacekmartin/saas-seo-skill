# JSON-LD schema.org

The highest-ROI SEO investment after `<title>` + `<meta description>`. Powers knowledge panel, FAQ accordions, breadcrumb SERPs, voice assistants, and LLM crawler topical understanding.

## Architecture

One shared Blade component: `resources/views/components/seo/json-ld.blade.php`.

Pages choose which schemas to emit:
```blade
<x-seo.json-ld :schemas="['organization', 'website', 'softwareApplication', 'itemList', 'glossary', 'person', 'faq']" />
```

Each schema is its own `<script type="application/ld+json">` block. Linked via `@id` anchors (`https://yourdomain.com/#organization`, `#website`, `#software`, etc.) so search engines can connect the graph.

## High-ROI schemas

### Organization (always)

Foundational entity. Anchor for `<link rel="author">` and seller references in offers.

```json
{
  "@context": "https://schema.org",
  "@type": "Organization",
  "@id": "https://yourdomain.com/#organization",
  "name": "Notdown",
  "url": "https://yourdomain.com",
  "logo": "https://yourdomain.com/icons/icon-512.png",
  "foundingDate": "2026",
  "foundingLocation": {
    "@type": "Place",
    "address": { "@type": "PostalAddress", "addressCountry": "CZ" }
  },
  "knowsLanguage": ["en", "cs"],
  "contactPoint": [
    { "@type": "ContactPoint", "contactType": "customer support", "email": "support@..." },
    { "@type": "ContactPoint", "contactType": "security", "email": "security@..." },
    { "@type": "ContactPoint", "contactType": "privacy", "email": "privacy@..." }
  ],
  "founder": { "@id": "https://yourdomain.com/#founder" }
}
```

### WebSite (always)

Declares the site + (optionally) a `SearchAction`.

```json
{
  "@type": "WebSite",
  "@id": "https://yourdomain.com/#website",
  "url": "https://yourdomain.com",
  "name": "Notdown",
  "publisher": { "@id": "https://yourdomain.com/#organization" }
}
```

Add `potentialAction` with `SearchAction` only if you have real site search.

### SoftwareApplication (for SaaS)

```json
{
  "@type": "SoftwareApplication",
  "@id": "https://yourdomain.com/#software",
  "name": "Notdown",
  "applicationCategory": "DeveloperApplication",
  "operatingSystem": "Web",
  "offers": [
    {
      "@type": "Offer",
      "priceCurrency": "USD",
      "price": "29",
      "priceValidUntil": "2026-12-31",
      "seller": { "@id": "https://yourdomain.com/#organization" },
      "priceSpecification": {
        "@type": "UnitPriceSpecification",
        "price": "29",
        "priceCurrency": "USD",
        "billingDuration": "P1M",
        "referenceQuantity": { "@type": "QuantitativeValue", "value": "1", "unitCode": "MON" }
      }
    }
  ]
}
```

Eligible for Google price snippets in SERP. Keep `priceValidUntil` updated annually.

### Person (founder, for E-E-A-T)

```json
{
  "@type": "Person",
  "@id": "https://yourdomain.com/#founder",
  "name": "Martin Macháček",
  "url": "https://yourdomain.com/about",
  "sameAs": ["https://github.com/machacekmartin"]
}
```

Boosts experience/expertise signals. Link `sameAs` to real verified profiles (GitHub, Mastodon, etc.).

### FAQPage (high-value when content matches)

```json
{
  "@type": "FAQPage",
  "@id": "https://yourdomain.com/#faq",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "...",
      "acceptedAnswer": { "@type": "Answer", "text": "..." }
    }
  ],
  "speakable": {
    "@type": "SpeakableSpecification",
    "cssSelector": ["[data-speakable]", "h1", "h2"]
  }
}
```

Notes:
- Google deprecated FAQ rich snippet for non-gov/health sites in 2023, but Bing + LLM crawlers still parse FAQPage. Worth shipping.
- `speakable` helps voice assistants (Google Assistant, Siri Reader Mode).
- Match `mainEntity` Q&A with visible body HTML using microdata (`itemscope itemtype="https://schema.org/FAQPage"`).

### DefinedTermSet (topical authority, AI-era killer feature)

Define your domain's vocabulary. Massive boost for LLM crawler retrieval + Google topical understanding.

```json
{
  "@type": "DefinedTermSet",
  "@id": "https://yourdomain.com/#glossary",
  "name": "Monitoring glossary",
  "hasDefinedTerm": [
    {
      "@type": "DefinedTerm",
      "name": "HTTP uptime monitoring",
      "description": "Periodic HTTP requests against a URL on a fixed schedule. ...",
      "termCode": "http_uptime"
    }
  ]
}
```

Pick 5–10 core terms your product/industry uses. Write 2–4 sentence definitions. Don't fluff — definitions should be quotable.

### ItemList (features, products, comparison)

For pages listing things (monitor types, integrations, plans, MCP clients):

```json
{
  "@type": "ItemList",
  "@id": "https://yourdomain.com/docs/mcp#mcp-clients",
  "name": "MCP-compatible clients",
  "numberOfItems": 7,
  "itemListElement": [
    { "@type": "ListItem", "position": 1, "item": { "@type": "SoftwareApplication", "name": "Claude", "url": "..." } }
  ]
}
```

### BreadcrumbList (for nested pages)

Eligible for SERP breadcrumb display under the result title.

### Article + WebPage (legal docs, blog posts)

For each legal page:
```json
{
  "@type": "Article",
  "headline": "Privacy Policy",
  "datePublished": "2026-06-10",
  "dateModified": "2026-06-10",
  "author": { "@id": "#organization" },
  "publisher": { "@id": "#organization" }
}
```

`Article.datePublished` + `dateModified` replace OG `article:*` tags.

### HowTo + TechArticle (docs, tutorials)

```json
{
  "@type": "HowTo",
  "name": "Connect Claude Code to Notdown via MCP",
  "step": [
    { "@type": "HowToStep", "name": "Generate token", "text": "..." },
    { "@type": "HowToStep", "name": "Add server config", "text": "..." }
  ]
}
```

## Validation strategy

Test that every JSON-LD block on every indexable page:
1. Is valid JSON (`json_decode` returns non-null, `JSON_ERROR_NONE`).
2. Has `@context: https://schema.org`.
3. Has `@type`.
4. Has required fields per type (Organization needs name/url/logo, etc.).

Pest test pattern:
```php
it('every JSON-LD block is valid with @context + @type', function () {
    foreach (['/', '/privacy', '/docs/...'] as $url) {
        $r = $this->get($url);
        preg_match_all('#<script type="application/ld\+json">(.*?)</script>#s', $r->getContent(), $m);
        foreach ($m[1] as $json) {
            $d = json_decode($json, true);
            expect(json_last_error())->toBe(JSON_ERROR_NONE);
            expect($d)->toHaveKey('@context')->toHaveKey('@type');
        }
    }
});
```

## What NOT to over-engineer

- Don't ship every schema "just in case." Each script tag = parse cost for crawlers and bytes for users.
- Don't fake reviews / aggregate ratings. Google penalizes detected fake structured data.
- Don't ship `Product` schema for SaaS — use `SoftwareApplication` (more specific).
- Don't ship `Article` schema for marketing landing pages — use `WebPage` + `SoftwareApplication`.
- Don't ship `LocalBusiness` if you're not actually a physical business.

## Editing JSON-LD

Use one Blade component (`<x-seo.json-ld>`) that takes a `:schemas` array. Page passes the relevant subset. Centralized so updates (new founder, changed pricing) hit one file.
