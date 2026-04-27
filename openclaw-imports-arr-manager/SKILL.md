---
description: Manage *arr suite (Sonarr, Radarr, Lidarr, Readarr, Bazarr) via their HTTP APIs — status, queue, calendar, wanted, add/remove, subtitle search.
tags: [sonarr, radarr, lidarr, bazarr, arr, automation]
---

# arr-manager Skill

## Description

Manage *arr suite applications via their HTTP APIs. All *arr apps share the same `X-Api-Key` auth header and `-k` for self-signed certs.

## Configuration

Credentials via config file at `~/.hermes/arr-config.json`:

```json
{
  "sonarr":  {"url": "https://sonarr.lambla.eu",  "apiKey": "4843b94c9cca4eedba415588ada4ecb8"},
  "radarr":  {"url": "https://radarr.lambla.eu",  "apiKey": "8f11d76e7c304e229e0975af50aa0eff"},
  "lidarr":  {"url": "http://localhost:8686",     "apiKey": "YOUR_LIDARR_API_KEY"},
  "readarr": {"url": "http://localhost:8787",     "apiKey": "YOUR_READARR_API_KEY"},
  "bazarr":  {"url": "https://bazarr.lambla.eu", "apiKey": "67588e1fa97c9709c6d31bc22912f07c"}
}
```

## Commands

- `status <app>` — System status (version, health, uptime)
- `list <app>` — Current download queue
- `upcoming <app> [days]` — Upcoming calendar (default 7 days)
- `wanted <app>` — Missing/wanted items
- `add <app> <title>` — Add show/movie by title
- `remove <app> <id>` — Remove item by ID
- `subtitles <app> [id]` — List subtitles for all movies or a specific one (Bazarr: list or trigger via UI)

## Usage

```
/arr-manager status sonarr
/arr-manager upcoming sonarr 14
/arr-manager wanted sonarr
/arr-manager list radarr
/arr-manager add sonarr "The Bear"
/arr-manager remove sonarr 12
/arr-manager search-subtitles bazarr 39
```

## API Reference

### Sonarr / Radarr / Lidarr / Readarr

Base: `${URL}/api/v3/` — responses are direct arrays/objects.

```bash
# Status
curl -s -k "${URL}/api/v3/system/status" -H "X-Api-Key: ${KEY}"

# Queue
curl -s -k "${URL}/api/v3/queue?page=1&pageSize=20" -H "X-Api-Key: ${KEY}"

# Calendar
curl -s -k "${URL}/api/v3/calendar?start=${START}&end=${END}" -H "X-Api-Key: ${KEY}"

# Wanted missing
curl -s -k "${URL}/api/v3/wanted/missing?page=1&pageSize=20" -H "X-Api-Key: ${KEY}"

# Add series by title (Sonarr)
curl -s -k -X POST "${URL}/api/v3/series" \
  -H "X-Api-Key: ${KEY}" -H "Content-Type: application/json" \
  -d '{"title":"Show Name","qualityProfileId":1,"tvdbId":0,"addOptions":{"searchForMissingEpisodes":true}}'

# Add movie by tmdbId (Radarr) — REQUIRES qualityProfileId + rootFolderPath
# Without qualityProfileId: "Path must not be empty" (misleading — needs the profile ID)
curl -s -k -X POST "${URL}/api/v3/movie" \
  -H "X-Api-Key: ${KEY}" -H "Content-Type: application/json" \
  -d '{"tmdbId":1417127,"qualityProfileId":1,"rootFolderPath":"/mnt/sdc/public/Films","monitored":true,"addOptions":{"searchForMovie":true}}'

# Alternative: look up tmdbId from movie title first
TMDBID=$(curl -s -k "${URL}/api/v3/movie/lookup?term=tmdb%3A${TMDBID}&apikey=${KEY}" | jq -r 'first | .tmdbId')

# Remove by ID
curl -s -k -X DELETE "${URL}/api/v3/series/${ID}" -H "X-Api-Key: ${KEY}"
```

### Bazarr

Base: `${URL}/api/` — responses wrapped in `{"data": [...]}`. **No `/v3/`**.

```bash
# List all movies
curl -s -k "${URL}/api/movies" -H "X-Api-Key: ${KEY}"
# Response: {"data": [{title, year, subtitlesCount, missing_subtitles, hasSubtitles, subtitles:[...], radarrId}, ...]}

# Get single movie (by radarrId)
curl -s -k "${URL}/api/movies/${RADARR_ID}" -H "X-Api-Key: ${KEY}"
# Subtitles array: [{name, code2, code3, path, forced, hi, file_size}]

# List series
curl -s -k "${URL}/api/series" -H "X-Api-Key: ${KEY}"

# Get single series (by sonarrId)
curl -s -k "${URL}/api/series/${SONARR_ID}" -H "X-Api-Key: ${KEY}"

# Bazarr Subtitle Search Trigger
**Note: Bazarr does not expose subtitle search via REST API.** The search is a UI-only action. Options:
1. Navigate to `/movies/{radarrId}` and click the **Search** button
2. Use the **Manual** button to search a specific language provider
3. Trigger via Bazarr's CLI inside the container: `docker exec bazarr bazarr.py --update-movie-subtitles -i {radarrId}`
4. Check job status via `GET /api/jobs`
```

### Bazarr Subtitle Fields

```
subtitlesCount   → total subtitle entries (embedded + external)
missing_subtitles → list of language codes with no sub found
hasSubtitles     → bool, true if ≥1 subtitle present
subtitles array:
  name      → "English", "French", etc.
  code2     → "en", "fr", etc.
  path      → null = embedded only, "/path/to/file.srt" = external file downloaded
  forced, hi → bool flags
  file_size → size in bytes (null if embedded)
```

## Notes

- Sonarr/Radarr v4 use `/api/v3/`, Bazarr uses `/api/` (no v3)
- Bazarr responses always wrapped in `{"data": ...}` — access `.data` key
- Use `-k` for self-signed certs on all *arr instances
- Radarr movie list: `GET /api/v3/movie?apikey=${KEY}` — direct array
- **Radarr movie add via tmdbId: `qualityProfileId` is mandatory** — without it, Radarr returns "Path must not be empty" even when `rootFolderPath` is set. Also requires `rootFolderPath` and `monitored:true`
- Sonarr calendar: `GET /api/v3/calendar?start=&end=` — ISO dates
- **Always prefer API over browser** — all actions are scriptable via curl
- **Bazarr subtitle search:** Confirmed 405 on `POST /api/movies/{id}/subtitle-search` — UI-only, no REST endpoint. Use browser or `docker exec bazarr bazarr.py --update-movie-subtitles -i {radarrId}`

## Damien's Services

| App | URL | Key |
|-----|-----|-----|
| Sonarr | `https://sonarr.lambla.eu` | `4843b94c9cca4eedba415588ada4ecb8` |
| Radarr | `https://radarr.lambla.eu` | `8f11d76e7c304e229e0975af50aa0eff` |
| Bazarr | `https://bazarr.lambla.eu` | `67588e1fa97c9709c6d31bc22912f07c` |
| Lidarr | not running | — |
| Readarr | not running | — |
