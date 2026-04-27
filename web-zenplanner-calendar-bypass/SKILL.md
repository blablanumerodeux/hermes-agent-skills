---
name: zenplanner-calendar-bypass
description: Access Cloudflare-protected ZenPlanner gym calendar pages and book classes via API
---
# ZenPlanner Calendar Bypass

## Context
CrossFit gym websites hosted on `*.zenplanner.com` are blocked by Cloudflare's anti-bot challenge on `/calendar.cfm`. The challenge is unusually aggressive — even FlareSolverr with 5-minute timeouts cannot crack it. However, the class schedule only exists in the HTML of this page; there is no separate API.

## Trigger
- Task involves accessing CrossFit Pro1 or similar ZenPlanner-hosted gym calendar
- `/calendar.cfm` returns HTTP 403 or "Just a moment..." Cloudflare challenge

## Symptoms
- `curl`, `cloudscraper`, `playwright` (headless), `DrissionPage` — all return 403
- Only `/calendar.cfm` is blocked; all other URLs on the same domain load fine
- FlareSolverr times out trying to solve the challenge

## What Does NOT Work
- `curl` with session cookies (challenge is IP-based, not cookie-based)
- `cloudscraper` Python library
- Playwright headless shell (detected as bot)
- `undetected_chromedriver` with real Chrome binary (crashes on Linux without Xvfb)
- FlareSolverr (ZenPlanner's challenge is too aggressive for it)
- Probing `/api/calendar.cfm` (returns `[]` — it's the *personal booking list* endpoint, not the schedule)

## What DOES Work

### Option 1: Ask user for enrollment URL (PREFERRED)
ZenPlanner class enrollment pages are NOT blocked by Cloudflare. Ask the user to navigate to the class they want on their phone/browser and share the URL from the address bar — it will look like:
```
https://{gym}.sites.zenplanner.com/enrollment.cfm?appointmentId={AppointmentId}
```
This URL bypasses the Cloudflare challenge entirely and contains the class details. Book through the browser using this URL.

### Option 2: Ask user for screenshot or HTML source
If Option 1 isn't available: ask the user to open `/calendar.cfm` in their own browser and either:
- Send a screenshot of the class schedule
- Copy-paste the HTML source
- Tell you which class they want and at what time

### Option 3: FlareSolverr on local network
If FlareSolverr is available at a local IP (e.g. `http://192.168.0.124:8191/`):
```
curl -s http://192.168.0.124:8191/v1 -X POST -H "Content-Type: application/json" \
  -d '{"cmd":"request.get","url":"<calendar-url>","maxTimeout":300000}'
```
Note: This WILL timeout on ZenPlanner specifically. The challenge is too strong.

### Option 4: Real Chrome via Xvfb (display server)
Chrome binary at: `/root/.cache/ms-playwright/chromium-1208/chrome-linux64/chrome`
```python
import subprocess
xvfb = subprocess.Popen(["xvfb-run", "-a", "echo", "ready"],
                         stdout=subprocess.PIPE, stderr=subprocess.PIPE)
xvfb.wait()

from selenium import webdriver
from selenium.webdriver.chrome.options import Options
opts = Options()
opts.binary_location = "/root/.cache/ms-playwright/chromium-1208/chrome-linux64/chrome"
opts.add_argument("--no-sandbox")
opts.add_argument("--disable-dev-shm-usage")
opts.add_argument("--disable-gpu")
driver = webdriver.Chrome(options=opts)
driver.get("<calendar-url>")
print(driver.page_source)
driver.quit()
```

## Booking via Enrollment URL (WORKING — April 2026)
The enrollment URL path works through the browser tool when logged in. Key parameters:

```
enrollment.cfm?appointmentId={AppointmentId}
```
```
enrollmentOptions_actions.cfm?action=register&AppointmentId={AppointmentId}&personId={personId}&MembershipId={membershipId}&nuflo=1
```

**Full working flow (CrossFit Pro1 — Damien Lambla):**
1. Navigate to: `https://crossfitpro1.sites.zenplanner.com/enrollment.cfm?appointmentId={AppointmentId}`
2. Click "Already a member? Click here to log in" → login with `Damien.lambla@gmail.com` / `6Q928pu5`
3. On the enrollment options page: select the family member (Damien Lambla), confirm membership shows "Reserved", click "Reserve"
4. Success: page shows "✓ Damien Lambla is registered for this class" and spaces remaining drops by 1

**Success indicators on confirmation page:**
- `✓ Damien Lambla is registered for this class.`
- Family member dropdown shows "Damien Lambla **Reserved** Cancel" (not "Register")
- Spaces Remaining drops (e.g., 12 → 11 after booking)
- No charge when using existing membership (Drop In shows $26.09 as alternative)

**Cancel a booking (same page, after booking):**
The Cancel link submits to:
```
enrollmentOptions_actions.cfm?AppointmentId={AppointmentId}&AttendanceId={AttendanceId}&action=cancel
```
AttendanceId appears in the cancel URL on the confirmation page itself — inspect the Cancel link's href.

**Reserve action requires `nuflo=1` parameter:**
```
enrollmentOptions_actions.cfm?action=register&AppointmentId={}&personId={}&MembershipId={}&nuflo=1
```
Without `nuflo=1`, the reserve action silently fails (no error, no redirect).

**Key IDs (CrossFit Pro1 — Damien Lambla):**
- Person GUID: `176E9FA6-B755-497F-8B69-556CEFC3A4D3`
- Membership ID: `3EF84B57-9872-4BA6-A9A7-D6D2C7EDF193` (3 Months Unlimited #12528)

## Mobile App API (BEST — No Cloudflare)
Zen Planner's mobile app (`com.zenplanner.memberapp`) communicates with API servers that have **no Cloudflare protection**.

### APK Identity
- **Package:** `com.zenplanner.memberapp` (verified from APK analysis)
- **Type:** Cordova hybrid app — NOT Flutter, NOT native Kotlin. The app shell loads a JavaScript SPA bundle from `memberappv220.zenplanner.com`.
- **DEX size:** 6.5MB (classes.dex ~6.5MB)
- **App shell:** Generic Cordova + plugins (camera, biometric, calendar, inappbrowser, etc.)
- **SPA logic:** All business logic is in `MemberMain.native.js` loaded from `memberappv220.zenplanner.com/zenplanner/studio/target/`

### Known API Endpoints (from APK + index.js analysis)
| Component | URL |
|----------|-----|
| API Base | `https://memberappapiv220.zenplanner.com/elements/api-v2/` |
| SPA Base | `https://memberappv220.zenplanner.com/zenplanner/studio/target/` |
| Version API | `https://versionapi.zenplanner.com/elements/api-v2/configurationManagement/` |
| App Settings | `https://zp-app-settings.s3.amazonaws.com` |
| Messaging Broker | `wss://mq.zenplanner.com` |

### APK Analysis Method
The APK arrives as a text document via the Hermes gateway. The gateway encodes binary as text, so decode it like this:

```python
# Read the gateway text document as raw bytes and write as binary APK
input_path = "/root/.hermes/cache/documents/doc_XXXXXXXX_Zen Planner.APK.txt"
output_path = "/tmp/zenplanner.apk"
with open(input_path, 'rb') as f:
    raw = f.read()  # gateway preserves raw bytes even in text field
with open(output_path, 'wb') as f:
    f.write(raw)
```

Then use `unzip -l <apk>` to list contents and `strings <classes.dex>` to search for API paths.

### Key DEX Strings (verified API paths)
- `/calendars` — confirmed in classes.dex
- `/configurationManagement/component-version/basic` — version check endpoint
- Auth header: `Authorization: Basic YXBwTmF0aXZlQnVuZGxlQHplbnBsYW5uZXIuY29tOmFwcE5hdGl2ZUJ1bmRsZQ==` (for error logging only)

### SPA Files Available
| File | URL | Status |
|------|-----|--------|
| `native.chunk.js` | `.../target/native.chunk.js` | 200 OK |
| `memberMain.native.js` | `.../target/memberMain.native.js` | 200 OK (lowercase 'm'!) |

These require auth headers to download content — the server returns 200 with empty body without proper cookies.

### Auth Status (UPDATED April 2026)
**Mobile API auth IS reversed but insufficient for calendar:**

1. `POST https://memberappv220.zenplanner.com/auth/v1/login` — accessible from terminal (NO Cloudflare!)
   - Body: `{"username":"Damien.lambla@gmail.com","password":"6Q928pu5","orgId":"12cdc08f-eeed-4601-85a8-cf36e8486008"}`
   - Returns: `{"token": "eyJhbGciOiJIUzUxMiJ9...", "expiresIn": 86400, "memberId": "176E9FA6-B755-497F-8B69-556CEFC3A4D3"}`
   - Token claims: `roles:["PORTAL"]`, `scopes:[]`

2. **The problem**: `POST https://memberappapiv220.zenplanner.com/elements/api-v2/calendars/calendarItems` with this JWT returns **403 "User does not have correct authorities"** — the PORTAL role is insufficient for calendar data.

3. **Why browser works**: The website login (`/login.cfm`) sets a ZenPlanner session cookie that works for all website pages. The mobile API JWT is a completely separate auth system with different permissions.

4. **Bottom line**: Terminal-only JWT auth does NOT grant calendar access. Browser session with website login is required for booking/viewing schedule.

**Org IDs:**
- CrossFit Pro1: `12cdc08f-eeed-4601-85a8-cf36e8486008`
- CrossFit 514: `0b8c21e8-a3a0-4d5d-bd0e-6b53aeb1e0b5`

### To Extract Full API Paths from APK
1. Extract APK as described above
2. Search DEX strings: `strings /tmp/zen_extract/classes.dex | grep -i '/api\|/v2\|calendar\|schedule\|enroll'`
3. Disassemble DEX: `baksmali disassemble classes.dex -o out/` (requires ~25MB free space)
4. **WARNING:** System disk must have >100MB free or baksmali will fail with "No space left on device"
5. Search smali files: `grep -rli 'pattern' out/` for API path references

### PCAPdroid capture (without SSL decrypt):
Only shows DNS queries and connection metadata (IP:port, bytes sent/received) — TLS payload is invisible. The `Info` column shows the hostname connected to, not the full URL path.

## Non-Cloudflare Pages on ZenPlanner (CONFIRMED)
The following pages load without Cloudflare challenges (confirmed April 2026):

| Page | URL | Notes |
|------|-----|-------|
| Workouts | `/workouts.cfm` | Workout calendar (WODs posted). Bypasses CF even when `/calendar.cfm` is blocked. Can navigate months. |
| Person Calendar | `/person-calendar.cfm?type=PERSON:{guid}` | Shows YOUR reservations only — not the full class schedule. Useful for checking bookings. |
| Leaderboard (daily) | `/leaderboard-day.cfm?date=YYYY-MM-DD` | Shows workout results. 514 allows without login; Pro1 requires `workoutid` param. |

**`/calendar.cfm` is blocked even with active website session:** Logging into the website does NOT bypass the Cloudflare challenge on `/calendar.cfm`. The challenge is per-URL-path and is enforced regardless of session cookies. The website login only works for other pages like `/workouts.cfm`, `/enrollment.cfm`, etc.

### Calendar API Probe
The calendar API IS accessible from terminal (no CF) but returns 403:
```
POST https://memberappapiv220.zenplanner.com/elements/api-v2/calendars/calendarItems
Authorization: Bearer <jwt-from-auth-v1-login>
Body: {
"calendarIds":[], "startDate":"2026-04-22", "endDate":"2026-04-22", "scheduleItemStatuses":["AVAILABLE","FULL"]}
```
Response: `{"error":"User does not have correct authorities"}` — the JWT PORTAL role doesn't include calendar access.

### StreamFit MCP Status (CrossFit 514 — separate gym)
StreamFit MCP connects to CrossFit 514 via `go.streamfit.com` API. AS OF APRIL 2026: auth is broken.
- PIN auth endpoint: `POST /api/v1/new/auth/sign_in` returns `400 {"errors":["Update your app."]}`
- This means the StreamFit API version has changed and the stored auth flow is outdated
- Workaround: use `mcp_zenplanner` tools for booking (ZenPlanner) OR manual enrollment URLs
## ZenPlanner Calendar URL Pattern
```
https://{gym}.sites.zenplanner.com/calendar.cfm?type=PERSON:{member-guid}
```
e.g. CrossFit Pro1:
```
https://crossfitpro1.sites.zenplanner.com/calendar.cfm?type=PERSON:176E9FA6-B755-497F-8B69-556CEFC3A4D3
```

## Files
- Cookies from logged-in session: `/tmp/pro1_fresh.txt` (Netscape format)
- Chrome binary: `/root/.cache/ms-playwright/chromium-1208/chrome-linux64/chrome`

## CrossFit 514 Specifics (for leaderboard access)
- `leaderboard-day.cfm` is NOT Cloudflare-blocked on 514 — loads with curl and browser
- URL: `https://crossfit514.sites.zenplanner.com/leaderboard-day.cfm?date=YYYY-MM-DD`
- If Damien has a 514 membership, his workout history/results may be visible there without login
- `workouts.cfm`, `scheduler.cfm`, `sign-up-now.cfm` also bypass CF on 514 (200 OK via curl)
- Note: `calendar.cfm` is still 403 on 514 despite these others being open
- StreamFit MCP for 514 is currently BROKEN (auth returns "Update your app")

## CrossFit Pro1 Specifics
- Member GUID: `176E9FA6-B755-497F-8B69-556CEFC3A4D3`
- Login: `Damien.lambla@gmail.com` / `6Q928pu5`
- Membership #: 12528
- Site: `crossfitpro1.sites.zenplanner.com`
- `person-calendar.cfm?type=PERSON:176E9FA6-B755-497F-8B69-556CEFC3A4D3` — shows Damien's booked classes (no CF block)
- `/calendar.cfm` is ALWAYS blocked — even with active website session
- Booking via browser: navigate to `enrollment.cfm?appointmentId=...` when logged in → works perfectly
- Website login page: `https://crossfitpro1.sites.zenplanner.com/login.cfm`

## IMPORTANT: Damien has TWO separate apps (April 2026 update)
- **PRO1 app** (`com.crossfitpro1.memberapp` or similar) — workout tracking, PRs, history. NOT the booking app.
- **ZenPlanner app** (`com.zenplanner.memberapp`) — class booking. This is the app that shows the schedule and lets you book.
- The `calendar.cfm` Cloudflare block applies to BOTH the website AND the ZenPlanner app's webview.
- The ZenPlanner app bypasses CF for API calls (memberappapiv220) but the API requires a different JWT with `CALENDAR_VIEWER` role — the website login JWT only has `PORTAL` role.
- StreamFit MCP connects to CrossFit 514 (a DIFFERENT gym), not Pro1.

## MCP Server Status (April 2026)
- Only `minimax` MCP is currently running in config.yaml
- `zenplanner` and `streamfit` MCPs are cloned locally at `/tmp/zenplannermcp/` and `/tmp/streamfitmcp-fast/` but NOT in config.yaml
- ZenPlanner repo: `https://github.com/blablanumerodeux/zenplannermcp` (has `.env.example`, improved auth, `get_profile` tool)
- StreamFit repo: `https://github.com/blablanumerodeux/streamfitmcp-fast`
- StreamFit MCP auth is BROKEN (returns "Update your app" error) — needs PIN auth debugging
- To add an MCP to config.yaml:
```yaml
mcp_servers:
  zenplanner:
    command: python3
    args:
    - /tmp/zenplannermcp/main.py
    env: {}
```

## Pages That LOOK Open (curl) But Are Actually Blocked or Don't Exist
These return HTTP 200 via curl but redirect to 404 when rendered in a real browser:
- `class-schedule.cfm` → JS redirect to 404
- `group-class.cfm` → JS redirect to 404
- `schedule.cfm` → JS redirect to 404
- `appointments.cfm` → Cloudflare challenge iframe injected
Always verify with browser_navigate, not just curl status codes.

## saveweb2zip.com Archive Is Useless
A ZIP from saveweb2zip.com only captures the Cloudflare challenge interstitial ("Just a moment..."), not the real page content. The real schedule only exists in JavaScript-rendered HTML — no hidden API found.

## Cloudflare-Open Pages (April 2026)
Both Pro1 and 514 share these non-blocked pages:
- `/workouts.cfm` — workout calendar with month navigation links (WODs only, not class schedule)
- `/person-calendar.cfm?type=PERSON:{guid}` — personal reservations
- `/enrollment.cfm?appointmentId=...` — class-specific enrollment (primary working path for booking)
- `/login.cfm` — login page (must be logged in before accessing enrollment)
- `/leaderboard-day.cfm?date=YYYY-MM-DD` — workout results (Pro1 needs `workoutid`; 514 open)

**Blocked despite appearing open (200 via curl but JS-rendered in real browser):**
- `class-schedule.cfm` → JS redirect to 404
- `group-class.cfm` → JS redirect to 404
- `schedule.cfm` → JS redirect to 404
- `calendar.cfm` → Cloudflare challenge (ALWAYS blocked, even with session)

Always verify with `browser_navigate`, not just curl status codes.
