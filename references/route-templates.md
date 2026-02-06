# Route Code Templates

## Table of Contents

- [namespace.ts](#namespacets)
- [Route Handler (.ts)](#route-handler-ts)
- [Route with Parameters](#route-with-parameters)
- [radar.ts](#radarts)
- [Common Patterns](#common-patterns)

## namespace.ts

```typescript
import type { Namespace } from '@/types';

export const namespace: Namespace = {
    name: '{SiteName}',
    url: '{domain.com}',
    description: '{Brief site description}',
    lang: '{en|zh-CN|ja|...}',
};
```

Fields: `name` (required), `url`, `description`, `lang`, `categories` (all optional).

## Route Handler (.ts)

```typescript
import { load } from 'cheerio';
import type { Route } from '@/types';
import cache from '@/utils/cache';
import got from '@/utils/got';
import { parseDate } from '@/utils/parse-date';

export const route: Route = {
    path: '/{route-path}',
    categories: ['{category}'],
    example: '/{namespace}/{route-path}',
    parameters: {},
    features: {
        requireConfig: false,
        requirePuppeteer: false,
        antiCrawler: false,
        supportBT: false,
        supportPodcast: false,
        supportScihub: false,
    },
    radar: [
        {
            source: ['{domain.com}/{page-path}'],
        },
    ],
    name: '{Route Display Name}',
    maintainers: ['{github-username}'],
    handler,
    url: '{domain.com}/{page-path}',
};

async function handler() {
    const rootUrl = 'https://www.{domain.com}';
    const currentUrl = `${rootUrl}/{page-path}`;

    const response = await got({
        method: 'get',
        url: currentUrl,
    });

    const $ = load(response.data);

    const items = $('{list-item-selector}')
        .toArray()
        .map((el) => {
            const $el = $(el);
            return {
                title: $el.find('{title-selector}').text().trim(),
                link: `${rootUrl}${$el.find('a').attr('href')}`,
                pubDate: parseDate($el.find('{date-selector}').attr('datetime')),
                author: $el.find('{author-selector}').text().trim() || undefined,
            };
        });

    const fullItems = await Promise.all(
        items.map((item) =>
            cache.tryGet(item.link, async () => {
                try {
                    const detailResponse = await got({
                        method: 'get',
                        url: item.link,
                    });
                    const content = load(detailResponse.data);
                    item.description = content('{article-body-selector}').html() || '';
                } catch {
                    // Keep list-page info if detail fetch fails
                }
                return item;
            })
        )
    );

    return {
        title: '{Feed Title}',
        description: '{Feed description}',
        link: currentUrl,
        image: '{logo-url}',
        item: fullItems,
    };
}
```

### Route interface fields

**Required:** `path`, `name`, `maintainers`, `handler`, `example`

**Optional:** `url`, `categories`, `parameters`, `features`, `radar`, `description`

- `path`: Hono route syntax — `:param` for required, `:param?` for optional
- `parameters`: `{ paramName: 'description' }` or `{ paramName: { description, default?, options? } }`

## Route with Parameters

```typescript
export const route: Route = {
    path: '/:category/:limit?',
    parameters: {
        category: {
            description: 'Content category',
            options: [
                { value: 'news', label: 'News' },
                { value: 'blog', label: 'Blog' },
            ],
        },
        limit: {
            description: 'Number of items',
            default: '20',
        },
    },
    // ...
    handler,
};

async function handler(ctx) {
    const category = ctx.req.param('category');
    const limit = ctx.req.param('limit') ? Number.parseInt(ctx.req.param('limit')) : 20;
    // ...
}
```

## radar.ts

```typescript
export const radar = [
    {
        title: '{Route Display Name}',
        source: ['{domain.com}/{page-path}'],
        target: '/{namespace}/{route-path}',
    },
];
```

Add multiple entries for multiple routes in the same namespace.

## Common Patterns

### Relative URL handling

```typescript
const link = href.startsWith('http') ? href : `${rootUrl}${href}`;
```

### Date extraction fallbacks

```typescript
const pubDate = $el.find('time').attr('datetime')
    ? parseDate($el.find('time').attr('datetime'))
    : undefined;
```

### Multiple content selector fallbacks

```typescript
const body = content('.article-body').html()
    || content('.post-content').html()
    || content('.entry-content').html()
    || '';
```

### JSON API extraction (for CSR sites)

```typescript
const apiUrl = `https://api.{domain.com}/v1/articles?page=1`;
const { data } = await got(apiUrl);
const items = data.results.map((item) => ({
    title: item.title,
    link: `${rootUrl}/article/${item.id}`,
    pubDate: parseDate(item.published_at),
    description: item.content_html,
    author: item.author?.name,
}));
```
