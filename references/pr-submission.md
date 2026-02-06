# PR Submission Guide

## Prerequisites

- GitHub CLI (`gh`) installed and authenticated
- A GitHub account

## Fork and Branch

```bash
# Fork the repo (first time only)
gh repo fork DIYgod/RSSHub --clone=false

# Add fork as remote
git remote add fork https://github.com/{your-username}/RSSHub.git

# Create feature branch
git checkout -b feat/{namespace}-{route-name}
```

## Commit

### Stage only your route files

```bash
git add lib/routes/{namespace}/
```

Do NOT stage formatting changes to other files. If `pnpm run format` modified unrelated files, restore them with `git checkout -- .` before staging.

### Commit message format

```
feat(route): add {SiteName} {route description}
```

Examples:
- `feat(route): add TCTMD conference news`
- `feat(route): add GitHub trending repos`
- `feat(route): add Hacker News best stories`

## Push

```bash
git push fork feat/{namespace}-{route-name}
```

## Create PR

### PR template (mandatory format)

The PR **must** include a `routes` fenced code block with the route path(s). PRs without this block are **automatically closed**.

```bash
gh pr create --repo DIYgod/RSSHub \
  --title "feat(route): add {SiteName} {description}" \
  --body "$(cat <<'EOF'
## Involved Issue / 该 PR 相关 Issue

Close #

## Example for the Proposed Route(s) / 路由地址示例

```routes
/{namespace}/{route-path}
```

## New RSS Route Checklist / 新 RSS 路由检查表

- [x] New Route / 新的路由
    - [x] Follows [Script Standard](https://docs.rsshub.app/joinus/advanced/script-standard) / 跟随 [路由规范](https://docs.rsshub.app/zh/joinus/advanced/script-standard)
- [x] Anti-bot or rate limit / 反爬/频率限制
    - [ ] If yes, do your code reflect this sign? / 如果有, 是否有对应的措施?
- [x] [Date and time](https://docs.rsshub.app/joinus/advanced/pub-date) / [日期和时间](https://docs.rsshub.app/zh/joinus/advanced/pub-date)
    - [x] Parsed / 可以解析
    - [x] Correct time zone / 时区正确
- [ ] New package added / 添加了新的包
- [ ] `Puppeteer`

## Note / 说明

{Brief description of what the route does and data sources.}
EOF
)"
```

### Checklist guidance

- **Script Standard**: Check `[x]` if code follows RSSHub conventions (proper imports, typed Route export, `cache.tryGet`, `parseDate`)
- **Anti-bot**: Check the first `[x]`. Check the sub-item only if the site actually has anti-bot measures AND your code handles them
- **Date and time**: Check all three if dates are parsed with `parseDate` (it handles timezone correctly)
- **New package**: Check only if you added a new npm dependency
- **Puppeteer**: Check only if the route uses Puppeteer

## After Submission

CI will run these checks:
- **ESLint** and **Lint**: Code style and quality
- **Vitest**: Automated route tests
- **Docker Build**: Ensures the route builds in production
- **CodeQL / DeepScan / CodeFactor**: Security and code quality scans

Expected behavior for fork PRs:
- Vercel preview may fail (authorization issue for forks — this is normal)
- Some Vitest runs on older Node versions may fail due to native module issues (not your fault)
- All lint, format, and main Vitest checks should pass

Wait for maintainer review. They may request changes — update your branch and push to update the PR.
