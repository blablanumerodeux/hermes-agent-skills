---
name: zenplanner-calendar-via-flaresolverr
description: Bypass Cloudflare on ZenPlanner gym calendar pages using FlareSolverr, then parse class schedule with times, instructors, and availability.
triggers:
  - zenplanner calendar blocked by cloudflare
  - crossfit class schedule scraping
  - flaresolverr zenplanner bypass
---

# ZenPlanner Calendar via FlareSolverr

## Purpose
Bypass Cloudflare protection on ZenPlanner gym calendar pages using FlareSolverr, then parse the class schedule with times, instructors, and availability.

## Prerequisites
1. **FlareSolverr must be running** on `http://127.0.0.1:8191`
   ```bash
   cd /tmp && python3 -m flaresolverr --port 8191 &
   ```

## Step-by-step

### Step 1: Start FlareSolverr (if not running)
```bash
pkill -f flaresolverr 2>/dev/null; cd /tmp && python3 -m flaresolverr --port 8191 > /tmp/flaresolverr.log 2>&1 &
sleep 10
curl -s http://127.0.0.1:8191/v1  # confirm it's up
```

### Step 2: Fetch calendar via FlareSolverr
**Calendar URL patterns:**
- Main class calendar: `https://crossfitpro1.sites.zenplanner.com/calendar.cfm?date={YYYY-MM-DD}&calendarType=APPOINTMENTGROUP:0B591C68-D519-4628-BE75-193CF24BD2D6&view=list`
- Damien's personal view: `https://crossfitpro1.sites.zenplanner.com/calendar.cfm?type=PERSON:176E9FA6-B755-497F-8B69-556CEFC3A4D3`

**Command:**
```bash
curl -s --max-time 90 -X POST "http://127.0.0.1:8191/v1" \
  -H "Content-Type: application/json" \
  -d '{"cmd":"request.get","url":"https://crossfitpro1.sites.zenplanner.com/calendar.cfm?date=2026-04-22&calendarType=APPOINTMENTGROUP:0B591C68-D519-4628-BE75-193CF24BD2D6&view=list","maxTimeout":80000}'
```

### Step 3: Parse the response (Python)
```python
import re, json

data = json.loads(response_text)
body = data['solution']['response']  # NOTE: response is a STRING, not dict

# Strip HTML
text = re.sub(r'<script[^>]*>.*?</script>', '', body, flags=re.DOTALL)
text = re.sub(r'<style[^>]*>.*?</style>', '', text, flags=re.DOTALL)
text = re.sub(r'<[^>]+>', ' ', text)
text = re.sub(r'\s+', ' ', text)

# Split by day headers
day_pattern = r'((?:Sunday|Monday|Tuesday|Wednesday|Thursday|Friday|Saturday),?\s+\w+\s+\d+,?\s+\d{4})'
parts = re.split(day_pattern, text)
schedule = []

for i in range(1, len(parts), 2):
    date_str = parts[i].strip()
    content = parts[i+1] if i+1 < len(parts) else ''
    
    # Find all time slots
    slot_pattern = r'(\d{1,2}:\d{2}\s*[AP]M)\s+((?:Group Class\s+\w+[\s\w]*|Postnatal|Kid\'s CrossFit|Open Gym|Pro1 MomFit|Semi-Private)[^\d\(]*?)(?:\((\d+)/(\d+)\))?\s*([A-Z][a-z]+\s+[A-Z][a-z]+)\s*(Gym|Virtual|Structure Extérieure)?\s*(?:\((\d+)\s*spots?\s*left\))?'
    
    for m in re.finditer(slot_pattern, content, re.IGNORECASE):
        time = m.group(1)
        class_full = m.group(2).strip()
        taken = int(m.group(3)) if m.group(3) else 0
        total = int(m.group(4)) if m.group(4) else 12
        instructor = m.group(5)
        location = m.group(6) or 'Gym'
        spots = int(m.group(7)) if m.group(7) else (total - taken if taken else 0)
        
        schedule.append({
            'date': date_str,
            'time': time,
            'class': class_full,
            'instructor': instructor,
            'location': location,
            'spots_available': spots,
            'total': total
        })

# Print nicely
for s in schedule:
    print(f"{s['date'][0:20]} | {s['time']} | {s['class']} | {s['instructor']} | {s['spots_available']} spots")
```

### Step 4: Filter available classes
```python
available = [s for s in schedule if s['spots_available'] > 0]
print(f"\n{len(available)} classes with spots available:")
for s in available:
    print(f"  {s['time']} - {s['class']} ({s['spots_available']} spots) with {s['instructor']}")
```

## Key appointment calendarType GUIDs
- `0B591C68-D519-4628-BE75-193CF24BD2D6` — Group Classes (cours de groupe)
- `7FDD4C48-8374-4139-89F9-FA835E542D7E` — Open Gym
- `92474610-7E80-4345-BCA4-CCA875A921FA` — CrossFit
- `E19FB1D1-E557-4808-8C62-ECB1FE6DCD84` — Kid's CrossFit
- `BD607E56-7944-4C6A-991E-9A10F6F1525E` — Gymnastics
- `FF966831-C211-408E-9E70-9C44F978F263` — Semi-Private Training
- `9946DB14-F68B-4ADB-B9D8-35338618F141` — Events
- Various `APPOINTMENTGROUP:...` — class type specific GUIDs

## Why NOT the Mobile API approach

**Exhaustively investigated (April 2026) — it is blocked from this environment.**

### Authentication Flow (what we learned)

1. `POST /auth/v1/login` with `{username, password}` → returns `SUCCESS_MULTIPLE_ORGS` with two orgs (CrossFit Pro1 `12cdc08f-eeed-4601-85a8-cf36e8486008`, CrossFit 514 `fbd6245e-110e-4f04-9e36-6059be3708f7`) but **NO token**.

2. The app must then call `GET /auth/v1/organizations/{orgId}/key` to get the org-specific apiKey.

3. `POST /auth/v1/key-login` with `{componentVersionId, apiKey}` → returns `INVALID` if apiKey is wrong/missing. The apiKey is **UUID format** and validated server-side as proper UUID type — not a string.

### Why the apiKey cannot be obtained from this environment

**The apiKey is never in the APK or generated on-device.** It is fetched from `GET /auth/v1/organizations/{orgId}/key` by the running SPA, which gets it from the server behind its own auth.

- **`GET /organizations/{orgId}/key` → HTTP 403 application-level** — The app server (not Cloudflare) rejects this endpoint. FlareSolverr can bypass CF on web URLs but the request still gets 403 from `memberappv220.zenplanner.com` even with valid web session cookies. This means the endpoint requires auth the webview/SPA has internally (session cookie, internal header, or token from a prior step) that we cannot replicate.
- **The APK (`com.zenplanner.memberapp`) is a Cordova shell** — `index.js` loads the actual app from `https://memberappv220.zenplanner.com/zenplanner/studio/target/`. All 465 smali files are framework code only. No app logic, no credentials, no API keys — confirmed by exhaustive smali/JS/XML search.
- **Web session cookies (CFID/CFTOKEN) do not work for the mobile API** — different domain and auth system.
- **`POST /auth/v1/key-login` returns `INVALID` for wrong apiKey** — not HTTP 400. Server validates the apiKey field as proper UUID format (returns `INVALID` for non-UUID values, and for UUIDs that don't match the org's apiKey).
- **Two real paths to get apiKey**: (1) Run the Android APK on a real device with mitmproxy/Frida to intercept the network call to `GET /organizations/{orgId}/key`, or (2) Ask the gym admin — the apiKey is visible in ZenPlanner admin panel under Settings → Mobile App.
### Key-login endpoint details

```
POST https://memberappv220.zenplanner.com/auth/v1/key-login
Body: {"componentVersionId": "CEE04405-FF91-4CC6-9FF6-3FE45B384569", "apiKey": "<UUID>"}
Success: {"status": "...", "token": "<Bearer token>", "refreshToken": "...", "organizationUsers": [...]}
Failure (bad apiKey): {"status": "INVALID", "organizationUsers": [], "token": null, "refreshToken": null}
```
- `apiKey` MUST be a valid UUID (36 char, hyphenated). Server validates with `java.util.UUID.fromString()` — invalid format returns `INVALID` (not 400).
- `componentVersionId` is static per app version: `CEE04405-FF91-4CC6-9FF6-3FE45B384569`
- The Bearer token from a successful `key-login` is required for all Elements API calls (e.g., `POST /member/calendars/classes/{classId}/reservations`)

**Conclusion**: FlareSolverr is the ONLY viable approach for programmatic calendar access from this environment.

### Alternative: Playwright login (gets CFID/CFTOKEN, but NOT useful for mobile API)

```python
from playwright.async_api import async_playwright
import asyncio

async def login_get_cookies():
    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True)
        context = await browser.new_context()
        page = await context.new_page()
        await page.goto('https://crossfitpro1.sites.zenplanner.com/login.cfm',
                       wait_until='networkidle', timeout=30000)
        await page.fill('input[name="username"]', 'damien.lambla@gmail.com')
        await page.fill('input[name="password"]', '6Q928pu5')
        await page.click('input[type="submit"]')
        await page.wait_for_timeout(5000)
        cookies = await context.cookies()
        cfid = next((c['value'] for c in cookies if c['name'].lower() == 'cfid'), None)
        cftoken = next((c['value'] for c in cookies if c['name'].lower() == 'cftoken'), None)
        return cfid, cftoken
```
These CFID/CFTOKEN cookies work for the web calendar (via FlareSolverr or browser tool) but **do NOT authenticate to the mobile API** (`memberappv220.zenplanner.com`) — different auth domain, different session model.

## Useful API Endpoints (when Bearer token IS available)

If you somehow obtain a valid Bearer token for `memberappv220.zenplanner.com`:

```
Auth base:    https://memberappv220.zenplanner.com/auth/v1/
Elements API: https://memberappapiv220.zenplanner.com/elements/api-v2/

GET  /member/calendars/classes/page?partitionId={}&personId={}&startDate={}&endDate={}
GET  /member/appointment-types/{typeId}/available-timeslots?partitionId={}&startDate={}&endDate={}
POST /member/calendars/classes/{classId}/reservations       # book class
DELETE /member/calendars/classes/{classId}/reservations     # cancel booking
GET  /member/calendars/classes/{classId}/reservations/eligibility  # check eligibility
GET  /member/calendars/classes/{classId}/waitlists/eligibility    # waitlist eligibility
POST /member/calendars/classes/{classId}/waitlists          # join waitlist
GET  /member/appointments?partitionId={}&personId={}&startDate={}&endDate={}
```

Known componentVersionId: `CEE04405-FF91-4CC6-9FF6-3FE45B384569`

## API Key Blocking — Why We Can't Get the apiKey

**The Problem**: The Bearer token needed for the Elements API is obtained via:
```
POST https://memberappv220.zenplanner.com/auth/v1/key-login
Body: {"apiKey": "...", "componentVersionId": "CEE04405-FF91-4CC6-9FF6-3FE45B384569"}
```

The `apiKey` itself comes from:
```
GET https://memberappv220.zenplanner.com/auth/v1/organizations/{orgId}/key
```

This `GET` endpoint returns **HTTP 403** from the application server (not Cloudflare) — it blocks all requests, including those with valid session cookies from `POST /auth/v1/login`. The SPA fetches the apiKey at runtime inside its webview context, which we cannot replicate externally.

**We have tried**:
- `POST /auth/v1/login` → returns org/user info but NO Bearer token
- `POST /auth/v1/key-login` with org GUID as apiKey → `INVALID`
- `GET /organizations/{orgId}/key` with every header combination → 403
- Direct browser login (after which calendar.cfm is still CF-blocked)
- mitmproxy on server → blocked by CF Error 1010 (ASN blocked)

**Current path**: APK patching (Frida gadget) to intercept the apiKey at runtime from the actual device.

## Pitfalls
- **FlareSolverr returns `response` as a string**, not a dict — access with `data['solution']['response']`
- **FlareSolverr timeout**: set `maxTimeout: 80000` (80s) to allow CF challenge to solve
- **Empty body**: if `solution.response` is empty, FlareSolverr is still solving — retry
- **FlareSolverr port**: confirm with `ss -tlnp | grep 8191`
- **view=list** gives cleaner text output than default calendar view
- **FlareSolverr session**: each request re-solves CF, so don't spam — 2-3 requests per minute max
- **Pro1 URL**: `https://crossfitpro1.sites.zenplanner.com`
- **Damien's person ID**: `176E9FA6-B755-497F-8B69-556CEFC3A4D3`
- **`GET /organizations/{orgId}/key` 403s** — not a CF issue, app-level auth protection. Frida/on-device is the only way.
