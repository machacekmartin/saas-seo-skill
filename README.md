# seo-foundation

A Claude Code skill encoding battle-tested SEO setup for a SaaS website. Distilled from a full audit that ripped out cargo-cult meta tags and rebuilt the layout / sitemap / static files around what crawlers and unfurl scrapers actually consume in 2026.

Most "comprehensive SEO checklist" posts ship 3× the tags real consumers read. This skill is the opposite — strip-by-default with reasoning per tag.

## What's inside

- **`SKILL.md`** — trigger conditions, core principles, workflows, quick reference.
- **`rules/architecture.md`** — 4-layout pattern (marketing / public / app / guest), indexable vs noindex.
- **`rules/meta-tags.md`** — every common head tag with keep/drop verdict + reasoning.
- **`rules/open-graph.md`** — OG + Twitter card strategy. Single PNG 1200×630, no WebP/AVIF alternates.
- **`rules/json-ld.md`** — schema.org foundation (Organization, WebSite, SoftwareApplication, Person, FAQPage, DefinedTermSet, ...).
- **`rules/locale.md`** — URL-prefix vs cookie/session. When `hreflang` lies.
- **`rules/behind-login.md`** — strip SEO meta from app + guest layouts.
- **`rules/static-files.md`** — robots.txt, sitemap.xml, llms.txt, security.txt, humans.txt, manifest.webmanifest.
- **`rules/og-images.md`** — 1200×630 PNG, one per slug, no locale suffix.
- **`rules/titles.md`** — brand-first em-dash, translation files, required slots.
- **`rules/ai-crawlers.md`** — LLM-era SEO. GPTBot allowlist, llms.txt, DefinedTermSet, speakable markers.
- **`rules/anti-patterns.md`** — 24 patterns to remove on sight, with grep query.

## Install

Drop this repo (or a symlink to it) under your project's `.claude/skills/`:

```bash
cd <your-project>
mkdir -p .claude/skills
git clone https://github.com/machacekmartin/saas-seo-skill.git .claude/skills/seo-foundation
```

Or as a submodule if you want version-pinning:

```bash
git submodule add https://github.com/machacekmartin/saas-seo-skill.git .claude/skills/seo-foundation
```

Claude Code auto-discovers the skill via the frontmatter in `SKILL.md`. Triggers cover most SEO tasks: head meta, OG, JSON-LD, robots/sitemap/llms, behind-login audits, OG image strategy, theme-color, webmaster verification, and more.

## Stack assumptions

The code examples assume **Laravel + Blade + Livewire**. Principles transfer cleanly to any stack; syntax does not. For Next.js, Astro, SvelteKit, Rails, etc., the architecture / decision frameworks / anti-patterns / sitemap structure all apply — adapt the templating layer.

| Project | How portable |
|---|---|
| Laravel + Livewire SaaS | 90% — direct copy with brand swap |
| Laravel app (non-Livewire) | 75% — Blade syntax mostly carries |
| Next.js / Astro / SvelteKit | 60% — principles transfer, ~40% code rewrite |
| WordPress / static site | 50% — principles + anti-patterns + static files; templating irrelevant |
| Internal-only B2B (no SEO) | 10% — only the behind-login stripping matters |

## Philosophy

**Cargo cult is the enemy.** Every meta tag has a cost (template noise, ongoing maintenance, lies to crawlers) and a benefit (signal to a real consumer). Default to drop; require justification to ship.

**Match architecture to indexability.** Pages split into three buckets — public-indexable, public-noindex, behind-login. Don't conditionally emit OG tags in a single mega-layout; split layouts by bucket.

**Don't lie to Google.** If locales aren't on distinct URLs, don't fake hreflang. If your X handle doesn't exist, don't ship `rel="me"` to it.

**Required slots > fallbacks.** Layouts that compute defaults hide bugs. Make slots required → Blade throws if missing. Discovery at render is cheaper than discovery in production.

## License

MIT.
