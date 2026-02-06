---
name: rsshub-route-developer
description: >
  Develop RSSHub routes from any website URL. Use when users want to create RSS feeds for websites,
  build RSSHub routes, or submit route PRs to the RSSHub project. Handles the full workflow:
  analyze target page, generate route code (namespace.ts, route handler, radar.ts), run local
  dev server and test, format/lint, and submit PR to DIYgod/RSSHub. Also use when users ask
  about RSSHub route development patterns, debugging route issues, or RSSHub project conventions.
---

# RSSHub Route Developer

Develop RSSHub routes that turn any website into an RSS feed. Follow the six phases below in order.

## Phase 1: Environment Setup

Check if an RSSHub repository clone exists in the working directory or a nearby path.

**If no RSSHub clone exists:**

```bash
git clone https://github.com/DIYgod/RSSHub.git
cd RSSHub
pnpm install
```

**If clone exists:** verify dependencies are installed (`node_modules/` exists). Run `pnpm install` if not.

Requirements: Node.js >= 22, pnpm (install with `npm install -g pnpm` if missing).

## Phase 2: Analyze Target Page

Before writing code, analyze the target website to determine the data extraction strategy.

1. **Fetch the page** and inspect the HTML structure
2. **Determine rendering type:**
   - **SSR (server-side rendered):** HTML contains article data directly — use `got` + `cheerio`
   - **CSR (client-side rendered):** HTML is mostly empty shells, data loaded via JS — check network requests for JSON API endpoints first; use Puppeteer only as last resort
3. **Identify key selectors** for: article list container, title, link, date/time, author, image (optional)
4. **Check detail pages:** if the list page only has titles/links, plan to fetch each article page for full content
5. **Note the date format** for `parseDate` compatibility

## Phase 3: Generate Route Code

Create three files under `lib/routes/{namespace}/`. See `references/route-templates.md` for complete templates and type definitions.

### File structure

```
lib/routes/{namespace}/
├── namespace.ts      # Site metadata
├── {route-name}.ts   # Route handler (main logic)
└── radar.ts          # Browser extension discovery rules
```

### Namespace naming

- Use the site's secondary domain as namespace (e.g., `tctmd` for tctmd.com)
- All lowercase, hyphens for multi-word names

### Route handler essentials

The handler function follows this pattern:

1. Build the target URL from route parameters
2. Fetch the list page with `got`
3. Parse HTML with `cheerio` (`load` from `'cheerio'`)
4. Extract item list (title, link, pubDate, author)
5. Fetch full article content per item using `cache.tryGet` for caching
6. Parse dates with `parseDate` from `@/utils/parse-date`
7. Return `{ title, link, description, image, item }` — the `item` array contains RSS entries

### Key imports

```typescript
import { load } from 'cheerio';
import type { Route } from '@/types';
import cache from '@/utils/cache';
import got from '@/utils/got';
import { parseDate } from '@/utils/parse-date';
```

### Route parameters

Access via Hono context: `ctx.req.param('name')` for path params, `ctx.req.query('name')` for query strings. Use `??` for defaults.

### Valid categories

`popular`, `social-media`, `new-media`, `traditional-media`, `bbs`, `blog`, `programming`, `design`, `live`, `multimedia`, `picture`, `anime`, `program-update`, `university`, `forecast`, `travel`, `shopping`, `game`, `reading`, `government`, `study`, `journal`, `finance`, `other`

### Features object

Set all to `false` unless the route specifically needs them:

- `requireConfig`: needs environment variables (API keys, etc.)
- `requirePuppeteer`: needs headless browser
- `antiCrawler`: site has anti-bot measures
- `supportBT` / `supportPodcast` / `supportScihub`: content type support

## Phase 4: Local Testing

1. Start the dev server:

```bash
pnpm run dev
```

This runs `tsx watch` on port 1200.

2. Test the route:

```bash
curl http://localhost:1200/{namespace}/{route-path}
```

3. **Verify output:**
   - Response is valid RSS XML
   - `<item>` entries exist with title, link, description
   - Dates parse correctly
   - Full article content appears in descriptions (not empty)
   - No error responses or empty feeds

4. **Debug common issues:**
   - Empty descriptions → wrong CSS selector for article body; inspect the actual detail page HTML
   - Stale cached data after fix → restart dev server to clear memory cache
   - Port 1200 in use → kill the existing process first
   - No items → check if the site is CSR (view-source vs rendered DOM)

## Phase 5: Code Quality

Run formatting and linting before committing:

```bash
pnpm run format
pnpm run lint
```

**Important:** `pnpm run format` (oxfmt) may reformat files across the repo. After running, use `git checkout -- .` to discard changes to files you didn't create, then `git add` only your route files.

## Phase 6: Submit PR

See `references/pr-submission.md` for the full PR submission workflow including:

- Fork and branch strategy
- Commit message format
- PR template (the `routes` code block is mandatory — PR auto-closes without it)
- Checklist items

### Quick summary

```bash
# Fork (first time only)
gh repo fork DIYgod/RSSHub --clone=false
git remote add fork https://github.com/{username}/RSSHub.git

# Branch, commit, push
git checkout -b feat/{namespace}-{route}
git add lib/routes/{namespace}/
git commit -m "feat(route): add {site} {description}"
git push fork feat/{namespace}-{route}

# Create PR
gh pr create --repo DIYgod/RSSHub --title "feat(route): add {site} {description}"
```
