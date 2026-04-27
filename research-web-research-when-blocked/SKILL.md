---
name: web-research-when-blocked
description: Web research workflow when Google/DuckDuckGo block with CAPTCHAs — use mmx search query as a reliable fallback, then verify via direct browser navigation to specific pages.
triggers:
  - "web search blocked by CAPTCHA"
  - "Google blocked me"
  - "DuckDuckGo blocked me"
  - "research when browsing is blocked"
---

# Web Research When Blocked

## Problem
Google and DuckDuckGo frequently block Hermes with CAPTCHAs. Browser direct navigation also gets blocked by many sites (Spotify, Genius, etc.).

## Solution: mmx search query

The MiniMax CLI `mmx` has a working web search that bypasses most blocks:

```bash
mmx search query "search query" --limit 10 --output json
```

This is reliable and returns structured results with titles, links, snippets, and dates.

## Workflow

1. **First try**: `mmx search query` with the research question
2. **For details**: Use browser navigation to specific pages that mmx found
3. **For long pages** (Wikipedia with 5000+ lines): The browser snapshot truncates. Options:
   - Navigate to a specific section anchor: `https://en.wikipedia.org/wiki/Page#Section_Name`
   - Use Wikipedia API with proper URL encoding
   - Use Apple Music or Spotify embedded pages for album tracklists (these sometimes work when direct links don't)

## Wikipedia API Tips

- URL encode special characters: `Rosal%C3%ADa` not `Rosalía`
- Page title spaces become underscores: `Lux_(Rosal%C3%ADa_album)`
- Example API call:
```bash
curl -s "https://en.wikipedia.org/w/api.php?action=parse&page=Lux_(Rosal%C3%ADa_album)&section=14&prop=wikitext&format=json"
```

## Install mmx-cli if needed

```bash
npm install -g mmx-cli
mmx auth status  # verify it works
```

## Greenhouse.io Job Boards
Cloudflare uses `job-boards.greenhouse.io/{company}` (NOT `boards.greenhouse.io`). Direct navigation to the old URL format returns 404. Always search first with `mmx search query "site:job-boards.greenhouse.io/{company} {location}"`.

## Sites that commonly block / work
- ❌ Google: blocks with CAPTCHA
- ❌ DuckDuckGo: blocks with CAPTCHA
- ❌ Spotify: page not found for direct artist links
- ❌ Genius: blocks with CAPTCHA
Cloudflare's own `/careers/` pages are blocked by Cloudflare's own bot protection (even the browser automation can't get through). Always find jobs via third-party job boards (Indeed, Glassdoor, RemoteLeaf, Greenhouse) via mmx search, then navigate to the original posting.
- ✅ Greenhouse.io: works but URL format changed — use `job-boards.greenhouse.io` not `boards.greenhouse.io`
- ✅ Wikipedia: works but long pages truncate in snapshots
- ✅ Apple Music: works for album/track info
- ✅ mmx search query: always works
