# AGENTS Guide

This document is for AI agents extending or integrating with the Business Gemini Pool project.

## What This Service Does
- Flask service that proxies Google Gemini Enterprise with OpenAI-compatible APIs.
- Manages multiple Google accounts with round-robin selection, cooldowns, and JWT/session refresh.
- Handles OpenAI-style file upload/list/delete plus chat with text + images; can return generated images via URL or base64.
- Ships a single-page admin console (`index.html`) for account/model/config management.

## Key Files
- `gemini.py`: Flask backend, account/session/JWT logic, OpenAI-compatible endpoints, admin APIs, image cache.
- `business_gemini_session.json.example`: Config template; copied to `business_gemini_session.json` on first run if missing.
- `index.html`: Admin UI (vanilla JS/HTML/CSS), talks directly to backend APIs.
- `chat_history.html`: Simple chat/history viewer (requires admin auth).
- `docker-compose.yml`, `Dockerfile`, `requirements.txt`: Deployment artifacts.

## Runtime & Setup
- Python 3.8+; install deps with `pip install -r requirements.txt`, then `python gemini.py`.
- Or `docker-compose up -d` after creating `business_gemini_session.json` (mounted into the container).
- `LOG_LEVEL` env var sets initial log filter (default INFO). `ADMIN_SECRET_KEY` can preseed the admin token secret.
- On start, `print_startup_info()` copies the example config if needed, loads config, prints proxy/account/model info, then runs on `0.0.0.0:8000`.

## Configuration (`business_gemini_session.json`)
- `proxy`: Proxy URL; `proxy_enabled` toggles usage.
- `image_base_url`: Optional override for image URLs in responses; defaults to request host.
- `image_output_mode`: `"url"` (default) adds `/image/<file>` links; `"base64"` inlines data URIs.
- `log_level`: DEBUG/INFO/ERROR for server logging (also persisted when set via API).
- `admin_password_hash`, `admin_secret_key`: Stored/admin auth; set automatically on first login if empty.
- `api_tokens`: List of API tokens allowed for OpenAI-compatible endpoints.
- `accounts`: List of Google accounts:
  - `team_id`, `secure_c_ses`, `host_c_oses`, `csesidx`, `user_agent`, `available`, optional `cooldown_until/reason`.
- `models`: Model metadata for `/v1/models` listing (id, name, description, context_length, max_tokens, enabled/pricing fields).

## Auth Model
- OpenAI-compatible routes require either `X-API-Token` or `Authorization: Bearer <token>`. Admin token also passes.
- Admin routes require `X-Admin-Token` or `Authorization: Bearer <admin token>` (also stored as `admin_token` cookie).
- `POST /api/auth/login`:
  - If no password set, first successful call stores the bcrypt hash in config.
  - Returns an HMAC-signed admin token (secret from config/env) and sets `admin_token` cookie.
- API tokens are persisted in config; helpers `load_api_tokens()` / `persist_api_tokens()` keep in-memory `API_TOKENS` in sync.

## Backend Architecture (`gemini.py`)
- Logging: Replaces `print` with `filtered_print` honoring global `CURRENT_LOG_LEVEL`. `set_log_level()` can persist to config.
- Account lifecycle (`AccountManager`):
  - Loads/saves config, tracks `accounts`, and per-index `account_states` (`jwt`, `jwt_time`, `session`, `available`, `cooldown_until/reason`).
  - Round-robin selection via `get_next_account()`, skipping disabled or cooling accounts.
  - Cooldown windows: auth errors 900s, rate limit 300s (or until next PT midnight), generic 120s. Marked in config for UI display.
  - `ensure_jwt_for_account()` refreshes JWT if older than 240s; `ensure_session_for_account()` creates a session via `create_chat_session()`.
- JWT/session creation:
  - `get_jwt_for_account()` calls `https://business.gemini.google/auth/getoxsrf` with cookies to fetch `keyId/xsrfToken`, builds HS256 JWT (`create_jwt()`).
  - Sessions created against `biz-discoveryengine.googleapis.com/v1alpha/.../widgetCreateSession` using `team_id`.
- File handling (`FileManager`):
  - In-memory map `openai_file_id -> {gemini_file_id, session_name, filename, mime_type, bytes, created_at}`.
  - Resets on process restart; `/v1/files` responses only reflect current memory.
- OpenAI-compatible APIs:
  - `GET /v1/models`: Returns config models or default `gemini-enterprise`.
  - `POST /v1/files`: Uploads file to Gemini (`upload_file_to_gemini`), stores mapping, returns OpenAI-style file object.
  - `GET /v1/files`: Lists current in-memory files; `GET/DELETE /v1/files/<id>` inspect/remove mappings.
  - `POST /v1/chat/completions`:
    - Accepts standard `messages` or `prompts` with inline `files` (base64).
    - Accepts OpenAI `file`/`file_id` references; resolves to Gemini fileIds via `FileManager`.
    - Optional `account_id` forces a specific account (skips rotation, but still obeys cooldown).
    - Uploads inline images first (`upload_inline_image_to_gemini`), then sends text + fileIds via `stream_chat_with_images`.
    - Response parsing pulls text plus generated images (`generatedImages`, `attachments`, `inlineData`), caches to `image/` unless base64 mode, builds OpenAI-style `content` with image URLs or data URIs.
    - `stream=true` returns SSE with a single delta chunk plus stop chunk (not incremental Gemini streaming).
    - Token usage is approximated via string lengths.
- Image/cache:
  - `image` dir is created at startup; `cleanup_expired_images()` prunes files older than `IMAGE_CACHE_HOURS` (1h) on each chat request.
  - `/image/<filename>` serves cached files with MIME sniffing; path traversal guarded.
- Proxy:
  - `get_proxy()` respects `proxy_enabled`; `check_proxy()` probes `https://www.google.com`.
- Admin/ops APIs:
  - `/api/status`: account counts + proxy status (requires admin).
  - Accounts: CRUD + toggle + cookie refresh + JWT test (`/api/accounts/*`).
  - Models: CRUD (`/api/models`).
  - Config: `GET/PUT /api/config`, import/export, logging level (`/api/logging`), proxy test/status, API token management (`/api/tokens`).
  - Static pages: `/` → `index.html`; `/chat_history.html` gated by admin auth.

## Frontend (`index.html`)
- Single HTML with inline CSS/JS; no build tooling.
- Uses `API_BASE='.'` and `apiFetch()` wrapper to attach `X-Admin-Token` from `localStorage['admin_token']`; 401 triggers login modal.
- Main flows:
  - `loadAllData()` loads accounts, models, config, status, log level, API tokens on page init and after edits.
  - Account tab: table render, add/edit/delete, toggle availability, test JWT, refresh cookies.
  - Models tab: add/edit/delete model entries.
  - Settings tab: proxy URL update, image output mode select, config download/upload, proxy test, theme toggle, log level picker.
  - Tokens tab: list/create/delete API tokens.
- UI extras: toast notifications, modal system, light/dark theme persistence, periodic `/api/status` check (30s).
- `chat_history.html` reuses theme vars to show chat transcript UI; served only to admins.

## Development Notes / Pitfalls
- File mappings live only in memory; if you restart, `/v1/files` and chat file references reset.
- Respect `account_manager.lock` when touching shared account/session state to avoid races.
- Cooldown logic writes markers to config; account availability in UI includes cooldown checks.
- `requests` calls use `verify=False`; if adding new HTTP calls, be explicit about SSL/timeout/proxy.
- `image_output_mode="base64"` skips disk caching; URL mode writes to `image/` and serves via `/image/...`. Keep this in mind if changing response format.
- SSE responses are single-chunk; true incremental streaming would require deeper changes to `stream_chat_with_images` and response generator.
- Avoid storing secrets in git; `business_gemini_session.json` is mounted/writable and should stay local.

## API Quick Reference
- OpenAI-compatible: `GET /v1/models`, `POST /v1/chat/completions`, `POST/GET/DELETE /v1/files`, `GET /image/<file>`, `GET /health`.
- Admin: `/api/status`, `/api/accounts*`, `/api/models*`, `/api/config` (`GET/PUT/import/export`), `/api/logging`, `/api/auth/login`, `/api/tokens*`, `/api/proxy/test|status`.
- Auth headers: `X-API-Token` (OpenAI routes) or `X-Admin-Token` (admin). Bearer tokens accepted equivalently.

## Useful Commands
- Run locally: `pip install -r requirements.txt && python gemini.py`.
- Docker: `docker-compose up -d` (ensure `business_gemini_session.json` exists in project root).
- Health check: `curl http://localhost:8000/health`.
- Sample chat: `curl -H "X-API-Token: <token>" -H "Content-Type: application/json" -d '{"model":"gemini-3-pro-preview","messages":[{"role":"user","content":"hello"}]}' http://localhost:8000/v1/chat/completions`.
