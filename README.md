# runlog

A natural-language running tracker. Describe a run in plain English and Claude logs it, scores the pace, and keeps a history with a pace-trend chart.

Single self-contained `index.html` — no build tools, no frameworks. Claude API for the language parsing, localStorage for offline-first persistence, and an optional AWS backend for cross-device sync.

**Live:** https://shortillj.github.io/runlog/

---

## What it does

- **Log runs by talking to it** — e.g. *"8km this morning in 44 mins, felt strong"*. Claude extracts distance, time, feel, and a one-line note, then computes pace.
- **Edit runs two ways:**
  - the green **✎** button on any history row opens a modal (date, feel, distance, time, notes)
  - or just say it in chat — *"that 8k was actually 9k"*, *"change my last run's feel to hard"* — and Claude targets the right run
- **Conversational memory** — the chat sends a rolling window of recent turns, so follow-ups and corrections have context.
- **Stats row** — total runs, total kilometres, average pace, best pace.
- **Pace-trend chart** — appears once there are 2+ runs (Chart.js, pace on a reversed axis so faster reads as higher).
- **Delete** runs with the **×** button (confirm dialog).

---

## Tech stack

| Piece | Choice |
|-------|--------|
| File | one `index.html`, GitHub Pages ready |
| AI | Claude API, model `claude-sonnet-4-5`, called direct from the browser (`anthropic-dangerous-direct-browser-access: true`) |
| Chart | Chart.js via cdnjs |
| Fonts | DM Mono (mono/headers), DM Sans (body) |
| Local storage | `runlog_runs_v1` (runs), `runlog_apikey_v1` (API key, shared across all tools) |
| Backend (optional) | API Gateway + Lambda + DynamoDB + CloudFormation, region `ap-southeast-2` |

---

## Setup

1. Clone the repo (GitHub Desktop → Clone, or `git clone`).
2. Open `index.html` in a browser — it runs as-is from `file://`.
3. Click **api key**, paste your Anthropic key. It's stored only in your browser's localStorage and sent only to the Anthropic API. Get one at console.anthropic.com → API Keys.
4. Start logging.

To publish: push to the `runlog` repo and GitHub Pages serves it automatically. Hard-refresh (`Ctrl+F5`) after a push to skip the cached version.

---

## Phase 2 — AWS backend (cross-device sync)

The tool works fully on localStorage alone. The AWS backend is the optional upgrade that syncs runs across devices.

Config lives at the top of `index.html`:

```javascript
const AWS_CONFIG = {
  API_URL: 'https://<your-api-id>.execute-api.ap-southeast-2.amazonaws.com/prod',
  API_KEY: '<your-api-gateway-key>'
};
```

If `API_URL` still contains the placeholder, the app silently stays in local-only mode. When configured, it boots from DynamoDB and falls back to local data if the network is unavailable (the sync dot by the wordmark shows green = synced, red = offline, grey = local-only).

### Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/runs` | load all runs |
| POST | `/runs` | save a run |
| DELETE | `/runs/{id}` | remove a run |

### Known constraint — editing and sync

There is **no `PUT` / update route**. A remote edit is therefore implemented client-side as **delete the old item, then re-post the updated one**. Local state updates first; the remote round-trip is best-effort and falls back to the offline indicator if it fails.

The small risk: if the delete succeeds but the re-post fails, the run survives locally but not in DynamoDB until the next successful sync. Fine for single-user use. The clean fix is a real `PUT /runs/{id}` route backed by a DynamoDB `UpdateItem` call — a tidy API Gateway + Lambda exercise for later.

### Infrastructure

Backend lives in the separate `run-log-aws` project (`infrastructure/cloudformation.yml`, `lambda/index.py`, `deploy.sh`). See the AWS troubleshooting log for the gotchas already solved (CORS preflight needs MOCK integration; Lambda permission `SourceArn` must use the `execute-api` namespace; API Gateway CloudWatch logging needs an account-level role).

---

## Security note (open item)

`AWS_CONFIG` currently ships the API URL and key inside `index.html`, which GitHub Pages serves publicly even from a private repo. Anyone viewing source can read, add, and delete runs via those three endpoints. Blast radius is bounded to run data and request quota — not the AWS account — but it's a real exposure.

Planned fix: put **Amazon Cognito** in front of API Gateway so the client authenticates and no static key ships to the browser. (IAM / auth / least-privilege — Cloud Practitioner security domain.)

---

## Changelog

- **Run editing** — ✎ modal per row, plus chat-driven corrections via an `edit` action in the system prompt.
- **Conversational memory** — rolling multi-turn context sent to the API; failed turns kept balanced so they can't corrupt the next request.
- **Pace guard** — divide-by-zero on a 0 km entry now shows `—` instead of `Infinity`.
