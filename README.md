# Anywear Virtual Try-On

Anywear is a Chrome Manifest V3 extension and embeddable virtual try-on widget. It detects clothing products on fashion websites, injects a fitting-room UI, captures camera frames, sends reference garments to the Decart streaming service, displays the returned WebRTC video, and records product/session events.

This repository is primarily browser-side JavaScript and HTML. It is **not currently a Python project**.

## Important migration truth

A Chrome extension cannot be rewritten entirely in Python and still run in the browser in the same way.

These parts must remain JavaScript in the extension:

- Chrome extension APIs such as `chrome.runtime`, `chrome.storage`, `chrome.identity`, and `chrome.tabs`.
- Content scripts that inspect and modify retailer pages.
- DOM events, Shadow DOM, iframe creation, drag-and-drop, and CSS.
- `getUserMedia`, `MediaRecorder`, `RTCPeerConnection`, WebSocket hooks, and WebRTC video.
- The popup and widget page.

Python is a good fit for a backend service. It can own token creation, queue management, image proxying, billing decisions, logging, moderation coordination, and database work. The browser extension should call that Python API over HTTPS.

The recommended target is therefore:

```text
Browser extension (JavaScript + HTML + CSS)
        |
        | HTTPS JSON API
        v
Python backend (FastAPI or Flask)
        |
        +-- authentication and installation IDs
        +-- Decart token/session integration
        +-- queue and billing APIs
        +-- image proxy and validation
        +-- event logging and database
```

Do not begin by translating every `.js` file line by line. First separate browser responsibilities from server responsibilities, then move one backend boundary at a time.

## Current data flow

1. Chrome loads `manifest.json`.
2. The background service worker in `background.js` handles extension-wide messages, identity, storage, onboarding, tab opening, uninstall behavior, and telemetry.
3. Content scripts run on retailer pages. `button-detector.js` and the site filters find product images and purchase controls.
4. `content.js` decides whether the page is relevant and injects or opens the widget.
5. `widget-core.js` creates and manages the widget lifecycle and camera permission state.
6. `widget.html` provides the widget DOM and CSS. Its script tags load the runtime in a strict order.
7. `widget.bundle.js` is the compiled Decart SDK UI and streaming implementation. Treat it as a dependency unless you have its source and build process.
8. `widget-init.js` adds session timers, overlays, recording hooks, billing UI, moderation behavior, and host/extension integration.
9. The widget sends images/session messages to the Decart service and receives a remote WebRTC stream.
10. `billing.js`, `logger.js`, and `background.js` send session and product events to the configured backend/analytics services.

## File-by-file guide

### Extension entry points and configuration

| File | What it does | Python migration status |
|---|---|---|
| `manifest.json` | Manifest V3 metadata, service worker, popup, content-script order, permissions, host permissions, and web-accessible resources. | Keep as JSON. Python cannot replace this browser manifest. |
| `background.js` | Service worker. Stores an installation ID, reads Chrome profile identity, handles popup messages, opens tabs/onboarding, sets uninstall behavior, and sends background telemetry. | Keep the thin Chrome API wrapper in JS. Move server-side telemetry processing to Python. |
| `config.js` | Shared runtime settings: API URLs, widget side, timeout/session limits, recording, and moderation options. | Keep browser-safe UI settings in JS. Move secrets and server policy to Python environment variables. |
| `rules.json` | Declarative extension rules, if enabled by the shipped extension configuration. | Keep as extension metadata; inspect before changing. |
| `_metadata/` | Chrome-generated extension verification metadata. | Generated/packaging output; do not hand-edit. |

### Page integration and product detection

| File | What it does | Python migration status |
|---|---|---|
| `content.js` | Main content script. Detects eligible pages, injects the widget, coordinates host-page events, and reports errors/events. | Keep in JS because it runs inside the retailer page. |
| `button-detector.js` | Finds add-to-cart/buy controls and platform-specific product actions. Supports Shopify, Salesforce Commerce Cloud, WooCommerce, Magento, and custom pages. | Keep in JS. Python cannot query a user's live DOM from an extension content script. |
| `per_website_garment_filter.js` | Chooses a site-specific garment filter and exposes common operations such as rejection, canonical image URLs, and page fixes. | Keep in JS; move only the policy data to Python later if useful. |
| `revolve.js` | Revolve-specific URL/style-code filter and page-fix module. | Keep in JS. |
| `ugg.js` | UGG-specific garment category and SKU filtering. | Keep in JS. |
| `guess.js` | Guess-specific path/category filtering. | Keep in JS. |

### Widget runtime

| File | What it does | Python migration status |
|---|---|---|
| `widget.html` | Full widget document: layout, video elements, overlays, buttons, billing iframe, recording recap, CSS, and script load order. | Keep as HTML/CSS. It is the browser UI. |
| `widget-core.js` | Large widget lifecycle module. Manages camera permission states, opening/closing, iframe/pane behavior, session state, and interactions with the bundled SDK. | Keep in JS. Refactor into modules before attempting any migration. |
| `widget-init.js` | Widget integration layer loaded after the bundle. Adds session inactivity handling, recap UI, billing hooks, moderation/session-end behavior, and extension/web host coordination. | Keep browser behavior in JS; move API calls behind a Python client later. |
| `widget.bundle.js` | Precompiled Decart SDK bundle. Owns much of the virtual try-on UI and streaming behavior. | Keep as an external/browser dependency unless source and build tooling are available. |
| `widget-prefetch.js` | Warms API/billing resources and handles extension image proxy URLs to reduce first-load latency. | Browser prefetch stays JS; backend caching can move to Python. |
| `connection-indicator.js` | Adds the active/inactive connection indicator and tracks WebRTC connection state. | Keep in JS. |
| `stream-quality.js` | Adjusts camera capture dimensions and canvas capture behavior for the model's expected landscape stream; monitors slow connections. | Keep media capture in JS. Python should receive server-side quality metrics, not raw camera frames unless explicitly designed for that. |
| `recording.js` | Uses a canvas and `MediaRecorder` to record the remote try-on video, assemble a Blob, download/share it, and upload session media. | Keep recording in JS. Python can receive uploads and store/process them. |

### Billing, logging, moderation, and fallback UI

| File | What it does | Python migration status |
|---|---|---|
| `billing.js` | Billing integration: installation/session identity, status refresh, billing directives, heartbeats, try-on balance, hosted billing iframe, and purchase-intent events. | Move billing rules, balance, and persistence to Python. Keep iframe/message handling in JS. |
| `logger.js` | Defines the categorized `decartLog`/timing helpers used by the widget. | Keep a small browser logger; central event ingestion belongs in Python. |
| `crash_reports.js` | Captures content-script errors and unhandled promise rejections and reports them through the tracking path. | Keep capture in JS; send reports to a Python endpoint. |
| `moderation-reset-attempt.js` | Hooks the widget WebSocket before the SDK loads, observes moderation messages/acks, optionally replays a nudged garment image, re-arms detection, and ends a failed session. | Keep the WebSocket/browser hook in JS. Put moderation policy and authoritative enforcement on the server. |
| `moderation-overlay.js` | Renders the fallback dressing-error overlay and hides/obscures the stream after an unrecoverable moderation result. | Keep UI in JS; Python can return the result and message code. |

### Popup and static assets

| File | What it does | Python migration status |
|---|---|---|
| `popup.html` | Toolbar popup UI with “Open fitting room” and “Tutorial” actions. | Keep as HTML. |
| `popup.js` | Sends popup actions to the background service worker and logs toolbar clicks. | Keep as JS; change its API endpoint only if the backend moves. |
| `icons/` | Extension icons used by the manifest and toolbar. | Keep unchanged. |
| `default_video.mp4` | Bundled fallback/media asset. | Keep if still needed by the browser UI. |
| `tryon-preview.gif` | Preview/loading asset. | Keep if still used; otherwise remove during packaging cleanup. |
| `shirt1.svg`, `shirt2.svg`, `shirt3.svg` | Garment illustration assets. | Keep as static assets or move to a web-hosted asset package. |

### Documentation

| File | Subject |
|---|---|
| `docs/drag-flip-on-hover.md` | Design/implementation notes for drag and hover flipping behavior. |
| `docs/moderation-reset-attempt.md` | Detailed moderation reset protocol, WebSocket interception, timing, telemetry, and server-side caveats. |
| `docs/per-website-garment-filters.md` | Per-site garment filtering behavior and extension constraints. |
| `docs/post-session-recap-ux.md` | UX rationale and behavior for the recorded post-session recap screen. |
| `docs/shadow-dom-image-scanning.md` | Notes on scanning product images inside Shadow DOM boundaries. |

## What Python should replace first

The cleanest first migration is a Python API, not a Python browser extension.

### Phase 0: establish a baseline

Before changing behavior:

1. Load the unpacked extension in `chrome://extensions` with Developer mode enabled.
2. Record the extension version and the API endpoints from `config.js`.
3. Test at least one supported fashion site and one unsupported site.
4. Test camera permission, a successful try-on, a queue response, session timeout, recording recap, billing panel, and moderation failure.
5. Save browser console/network logs for each flow.
6. Write down the current JSON request and response shapes. Do not guess them from variable names.

There is no `package.json`, Python environment file, or test runner in this repository, so the current project does not expose a reproducible local build/test command. Add those as part of the migration.

### Phase 1: create the Python service

Recommended stack:

- FastAPI for typed HTTP endpoints and automatic OpenAPI documentation.
- Pydantic models for request/response validation.
- `httpx` for outbound HTTP calls to Decart and other services.
- PostgreSQL for installations, sessions, billing state, and events.
- Alembic for database migrations.
- `pytest` and `pytest-asyncio` for tests.
- `uvicorn` for local development.

Start with a small service layout:

```text
server/
  app/
    main.py              # FastAPI application and route registration
    config.py            # environment variables, never hard-coded secrets
    models.py            # Pydantic request/response models
    dependencies.py      # database/auth/request dependencies
    routes/
      health.py
      tokens.py
      queue.py
      images.py
      billing.py
      events.py
    services/
      decart.py           # outbound Decart client
      billing.py          # balance and billing decisions
      telemetry.py        # event normalization and persistence
    db/
      session.py
      tables.py
  tests/
```

Example first endpoint:

```python
# server/app/main.py
from fastapi import FastAPI

app = FastAPI(title="Anywear API")

@app.get("/health")
def health() -> dict[str, str]:
    return {"status": "ok"}
```

Run it locally with:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install fastapi uvicorn[standard] pydantic-settings httpx pytest
python -m uvicorn app.main:app --reload --app-dir server
```

### Phase 2: move configuration and token creation

1. Add a Python `/api/tokens` route.
2. Move the Decart secret/API credential to a server environment variable.
3. Validate the browser request with a Pydantic model.
4. Keep the response shape compatible with the current `widget.bundle.js` integration.
5. Change `DECART_CONFIG.TOKEN_ENDPOINT_URL` to the Python service URL.
6. Verify that no secret is shipped in the extension.

A browser request should contain an installation/session identifier and the minimum required try-on parameters. The server should validate ownership, rate limits, billing, and origin before requesting a provider token.

### Phase 3: move image proxying

Implement `/api/proxy-image` in Python using `httpx` only after adding protections:

- Allow only `http` and `https` source URLs.
- Enforce an allowlist or strict redirect policy.
- Set connection, read, and total size limits.
- Validate content type and image bytes.
- Avoid returning arbitrary response headers.
- Prevent access to localhost, private IP ranges, cloud metadata endpoints, and internal network addresses.
- Add caching and a maximum image size.

This endpoint is security-sensitive because it fetches a URL supplied by a browser page.

### Phase 4: move queue, billing, and telemetry

Move one contract at a time:

1. `/api/queue` or the existing queue endpoints.
2. `/api/status` for try-on balance and billing state.
3. Billing heartbeat/session accounting.
4. `/api/log` event ingestion.
5. Upload endpoints used by recording/session recap.

Keep the browser event names stable during the migration. Replace implementation behind the API first; rename events only after dashboards and consumers are updated.

### Phase 5: improve the browser code without translating it

After the Python API is stable:

- Replace minified one-file modules with readable ES modules.
- Add a `src/extension/` directory and keep `widget.html` as the browser entry point.
- Add a build tool such as Vite or esbuild only if needed.
- Add browser tests with Playwright for widget states.
- Add contract tests that run against a local Python API.
- Keep `widget.bundle.js` isolated behind a small adapter.

## Suggested Python learning order

Follow this order so every lesson has a visible result:

1. Python syntax: variables, strings, lists, dictionaries, conditions, loops, and functions.
2. Type hints and Pydantic models: represent the JSON currently exchanged by the extension.
3. HTTP basics: methods, status codes, headers, JSON, timeouts, and retries.
4. FastAPI routes: build `/health`, then `/api/tokens` with a fake provider.
5. Async programming: understand `async def`, `await`, and concurrent outbound requests.
6. Database fundamentals: installations, sessions, events, and migrations.
7. Authentication and authorization: identify an installation without trusting arbitrary browser input.
8. Security: SSRF protection for image proxying, CORS, rate limits, secret management, and input validation.
9. Testing: unit tests for services, API tests for routes, and contract tests for the extension.
10. Deployment: environment variables, HTTPS, logging, monitoring, migrations, and rollback.

For every feature, use this learning loop:

```text
Read the existing JS contract
      -> write the JSON model in Python
      -> implement a fake/local endpoint
      -> add a test
      -> point one browser call at it
      -> test the real flow
      -> only then move the next endpoint
```

## What not to move to Python

Do not send camera frames to Python merely because the goal is “use Python.” That would add latency, bandwidth, privacy risk, and a new media pipeline while the browser already has the correct WebRTC and MediaRecorder APIs.

Do not put provider secrets in `config.js`, `widget.html`, `popup.js`, or any other shipped extension file. Anything shipped to Chrome can be inspected by the user.

Do not edit `widget.bundle.js` directly unless you own its source/build process. Add an adapter around it and preserve the integration contract.

Do not trust `installation_id`, `session_id`, price, balance, moderation result, or origin values supplied by the browser. The Python server must derive or verify authoritative values.

## First practical task

The safest first implementation task is:

1. Create `server/app/main.py` with `/health`.
2. Add `server/tests/test_health.py`.
3. Run the test locally.
4. Add a development-only `/api/tokens` fake response with a typed Pydantic request.
5. Point a copied development config at that endpoint.
6. Confirm the browser reaches Python before integrating the real Decart provider.

Do not remove the existing JavaScript files during these steps. Keep the current extension working while the new backend is introduced incrementally.

## Current limitations and risks

- The repository contains minified or bundled JavaScript, so some internal behavior cannot be safely explained from this checkout alone.
- The exact backend API contracts are external to this repository and must be captured from network requests or provider documentation before rewriting.
- `manifest.json` includes a long extension key and the JavaScript files include analytics/provider configuration. Review shipped credentials and rotate anything that is actually secret.
- There is no visible automated test/build configuration in the root. Add repeatable checks before a broad migration.
- The `widget.html` script order is part of the runtime contract. In particular, `moderation-reset-attempt.js` must load before `widget.bundle.js`, and `billing.js` must load before `widget-init.js`.

## Definition of done for the migration

The migration is not complete when Python merely starts. It is complete when:

- The extension still loads under Manifest V3.
- Camera permission and WebRTC try-on behavior remain in the browser.
- Python owns authenticated API calls, provider credentials, queue/billing decisions, persistence, and telemetry.
- Every moved endpoint has a documented request/response model.
- Unit, API, and browser contract tests pass.
- Secrets are absent from shipped extension files.
- The old backend path can be disabled or rolled back without republishing a broken extension.
