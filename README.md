# NEXUS — AWAARA AI Chief of Staff

A personal, local-first AI backend: one memory, one job queue, one audit trail, many interfaces. You talk to it by text or voice note (English, Hindi, Hinglish) and it tracks tasks, projects and reminders, writes documents, summarises read-only email, keeps a project registry, and sends reminders and briefings.

- **Local inference only.** Language understanding is a local Ollama model; speech-to-text is local faster-whisper. There is no paid API and no cloud fallback in the code (cloud-tagged and remote models are refused at startup; see `docs/SECURITY.md`).
- **₹0 recurring cost.** Storage is SQLite. No hosted service is required.
- **Safe by construction.** Single-owner authorization per channel; approvals bound to the exact action (single-use, expiring); quick internal saves are undoable; nothing external (send, publish, delete, submit) happens without an approval.
- **Status honesty.** This README only claims what has been run. `docs/STATUS.md` is the single source of truth and labels every feature TESTED / UNVERIFIED / MOCKED / MISSING / BLOCKED. History and decisions are in `docs/MILESTONES.md`.

## What works today

| Area | State | Evidence |
|---|---|---|
| Telegram (private bot, single owner, long polling) | Live-tested by the owner (text, voice, reminders, Undo, restart persistence). Rejection of strangers is covered by automated tests only, not yet tried live | `tests/e2e/test_telegram.py`, `docs/MILESTONES.md` M7 |
| Natural conversation (English / Hindi / Hinglish), grounded in your own tasks and notes, short private chat memory | Tested with a fake model and spot-checked on the real local model | `tests/integration/test_conversation.py`, `tests/e2e/test_real_local.py` (`-m slow`) |
| Voice notes: local Whisper -> local model | Tested on synthetic speech and on real notes through Telegram. Hindi/Hinglish accuracy is the weakest stage | `docs/benchmarks/`, `tests/e2e/test_real_local.py` |
| Act + Undo for quick saves (task, reminder, note); confirmation when input is uncertain | Tested | `tests/integration/test_act_undo.py` |
| Uncertain voice (low or unknown speech score) is read back, never executed from a guess | Tested | `tests/integration/test_voice_intent_regressions.py`, `test_voice_display.py` |
| Project status questions: project chosen by code from your registry; unclear -> asks which project | Tested | `tests/integration/test_project_status.py` |
| Tasks, projects, memory, search, priorities, reminders, reports; applications, startup tracker, content pipeline | Tested | unit + integration suites |
| Approvals (hash-bound, single-use, expiring, owner-only), safe file writing | Tested | adversarial tests |
| Always-on on macOS (LaunchAgent + watchdog) | Installed and running on the author's Mac. Start-after-reboot, crash-restart on the real instance and a real watchdog alert are not yet verified | `docs/OPERATIONS.md` |
| Channel layer (shared core, per-channel owner identity, typed yes/no/undo, shared outbox sender) | Tested with a fake second channel | `tests/integration/test_channels.py` |
| WhatsApp | Not connected. Only the channel interface and a legacy self-chat webhook parser (unverified, disabled by default) exist. Transport not chosen | `docs/WHATSAPP.md` |
| Gmail (read-only) / Notion | Mocked providers only; no real account connected | tests |
| Alexa | Request verification tested on signed fixtures; not tested on a real device | `tests/e2e/test_alexa.py` |
| Mac file agent | Tested on loopback only | tests |
| VPS deployment, remote execution agents | Plans / Docker files only; nothing deployed | `docs/DEPLOYMENT.md`, `docs/REMOTE_AGENTS.md` |

## Requirements

- macOS on Apple Silicon is what it is developed and run on (16 GB RAM). Other platforms are untested.
- Python 3.11 (`requires-python >=3.11,<3.12`).
- Ollama running on loopback with the local models: `qwen3.5:4b` (primary), `granite4.1:3b` and `qwen3:8b` (fallbacks). Roles are set with `COS_MODEL_*`; cloud models (`:cloud` / `-cloud`) are refused.
- `ffmpeg` (Homebrew: `/opt/homebrew/bin/ffmpeg`) for converting voice notes. The service also looks in `/opt/homebrew/bin` and `/usr/local/bin` because launchd runs with a minimal `PATH`.
- faster-whisper `small` model for speech: fetched once, explicitly, with `python scripts/fetch_whisper_model.py small` (about 460 MB; it is never downloaded silently at runtime).
- About 15 GB of free disk.

## Install and run locally

```bash
python3.11 -m venv .venv && source .venv/bin/activate
pip install --require-hashes -r requirements.lock        # runtime, hash-pinned
pip install --require-hashes -r requirements-dev.lock    # tests / lint (optional)
pip install -e . --no-deps

cp .env.example .env            # then fill values LOCALLY; .env is git-ignored. Never paste secrets into chat or issues.
python -m cos configure-telegram   # hidden-prompt entry of your BotFather token and your numeric id (written to .env, never echoed)
python -m cos telegram-check       # read-only verification, sends nothing
python -m cos health               # database, Ollama, speech model, disk, heartbeat
python -m cos run                  # worker + scheduler + Telegram + loopback API (127.0.0.1:8787)
```

Create the bot with @BotFather as a new dedicated bot (two processes polling one token conflict). `python -m cos setup-id` prints your numeric Telegram id locally once you message the bot.

### Environment variables

`.env.example` lists every setting with empty or safe defaults and no real credentials. The important ones: `TELEGRAM_BOT_TOKEN`, `TELEGRAM_ALLOWED_USER_ID`, `COS_MODEL_*`, `COS_WHISPER_MODEL`, `COS_CONFIRM_POLICY` (`act_undo` default, or `confirm_all`), `COS_VOICE_CONFIRM_BELOW` (default 0.6), `COS_CONVO_KEEP_TURNS` / `COS_CONVO_RETENTION_DAYS`, `COS_INDEX_ROOTS` (read-only workspace index; explicit folders only), `COS_DASHBOARD_TOKEN` (enables the private read-only dashboard).

## Always-on (macOS)

`deploy/launchd/` holds LaunchAgent templates (service, watchdog, Mac agent). They contain the author's home-directory paths: edit them for your machine before installing. Details and limits (lid closed, logout, reboot): `docs/OPERATIONS.md`.

## Tests and health checks

```bash
pytest                                  # fast suite: no network, no real model (553 tests at the time of this commit)
pytest -m slow -s                       # real Ollama + real Whisper tests (needs both available; 18 tests)
ruff check src tests scripts && mypy src
python -m cos health                    # runtime health
curl -s 127.0.0.1:8787/healthz          # liveness (no data)
python scripts/smoke_service.py         # sandboxed start/stop/crash-restart smoke test
```

Evidence and gates: `docs/TESTING.md` (its older figures are historical; the commands above are current).

## Security and privacy

- Inference and transcription are local. No message text or audio leaves the machine.
- Only the numeric owner id(s) in identities are accepted, per channel; strangers are dropped silently and audited. Ownership is channel-qualified (the same digits on another channel are a stranger).
- Voice audio is deleted right after transcription. Chat memory lives in a separate private file (`data/conversations.db`, mode 600, secure-delete, short retention, never in backups or logs).
- The model only classifies or writes text; code decides what runs. Uncertain voice is never executed from a guess. Consequential actions need a hash-bound, single-use approval; voice can never approve an external action.
- Secrets live only in `.env` (git-ignored). Never commit `.env`, `data/`, `backups/`, `logs/`, `profile/`, `workspace/`, databases, audio or model weights; `.gitignore` excludes them.

Full model: `docs/SECURITY.md`.

## Backup and recovery

- The scheduler writes a verified (integrity-checked) SQLite backup daily to `backups/` and keeps 14. Manual: `python -m cos backup`.
- Restore: stop the app, `python -m cos restore <backup> --to data/restored.db`, move the old DB aside, copy the restored file into place (`docs/OPERATIONS.md`).
- Backups contain your personal data. Keep them out of Git and out of shared storage. This repository holds code, tests and documentation only.
- Take a backup before upgrading: schema migrations run automatically at start.

## Known limitations

- Speech recognition of Hindi/Hinglish is imperfect; the speech score is an uncalibrated model value (not the probability a transcript is correct), so correct notes can still be read back for checking.
- The assistant does not yet remember a pending clarifying question: after "Which project?" you must restate the full request.
- Local 3-8B models can blur dates or wording in free conversation; dates for reminders are resolved by deterministic code, not by the model.
- One owner, one Mac. WhatsApp, real Gmail/Notion/Alexa, remote machines and VPS deployment are not connected.
- Reboot/crash recovery on the real service and a real watchdog alert have not yet been verified.

## Layout

```
src/cos/        config, db, services (domain logic), core (gateway, pipeline, actions, worker, scheduler, text decisions),
                ai (intents, conversation, router), speech, files,
                adapters (telegram, base/outbox sender, alexa, whatsapp), integrations (gmail, notion), api, agent
migrations/     0001 core · 0002 modules · 0003 NEXUS registry/capabilities · 0004 undo
deploy/         Dockerfile, compose files (isolated, never deployed), launchd templates, Alexa models, Traefik proposal
docs/           STATUS, MILESTONES, PLAN, ARCHITECTURE, SECURITY, OPERATIONS, TESTING, DEPLOYMENT,
                CONVERSATIONAL_PLAN, WHATSAPP, REMOTE_AGENTS, benchmarks/
scripts/        model/speech benchmarks, Whisper fetch, Gmail auth, service smoke test, watchdog
profile.example/  template for the CV/portfolio facts used for application drafts (copy to ./profile, which is git-ignored)
```

## Telegram commands

`/help` lists everything. Natural text and voice work without commands; examples: `/add`, `/list`, `/remind`, `/done`, `/undo`, `/clearchat`, `/project`, `/approvals`, `/health`.
