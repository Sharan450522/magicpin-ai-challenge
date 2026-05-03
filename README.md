# Vera message engine (magicpin AI challenge)

Deterministic **`compose(category, merchant, trigger, customer?)`** plus a **FastAPI** bot exposing `GET/POST /v1/*` for the judge harness.

## Approach

- **Hybrid composition:** [Groq](https://console.groq.com) `llama-3.3-70b-versatile` (primary) and `llama-3.1-8b-instant` (fallback) with `temperature=0` and a fixed `LLM_SEED` for repeatability. If the API is empty/unavailable, **deterministic templates** in `templates.py` run (no API key required).
- **Grounding:** `signals.py` builds a sorted **facts** JSON from the four contexts only—no invented numbers, offers, or citations.
- **Playbooks:** `playbooks.py` dispatches by `trigger.kind` (send-as, CTA shape, must-include).
- **Validation:** no URLs, taboo words, `send_as` must match customer vs merchant scope; failed validation → one LLM repair attempt → template fallback.
- **Replies:** `conversation.py` handles auto-reply streaks, hostile opt-out, basic intent commitment (“let’s do it”), and off-topic decline.

## Tradeoffs

| Choice | Why |
|--------|-----|
| Groq | Fast + cheap; good JSON adherence with `response_format` where supported |
| Templates fallback | Guarantees uptime on `/v1/tick` within the 30s judge timeout |
| In-memory store | Matches brief; no Redis in starter repo |

## Setup

```powershell
cd magicpin-ai-challenge
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
copy .env.example .env
# Edit .env — set LLM_API_KEY (Groq) for LLM path
```

## Run locally

```powershell
uvicorn bot:app --host 0.0.0.0 --port 8080
```

**Public URL (ngrok):** `ngrok http 8080` — submit the `https://…` base URL (judge calls `https://host/v1/…`).

**Cloud:** build with `Dockerfile`; deploy on Render using `render.yaml` (set `LLM_API_KEY` as a secret).

## Dataset & submission

```powershell
python scripts/expand_dataset.py
python scripts/build_submission.py
```

Produces `expanded/` and `submission.jsonl` (30 lines from `expanded/test_pairs.json`, template path if no key).

## Judge simulator

1. Start the bot (`uvicorn`).
2. Edit **top of** `judge_simulator.py`: `BOT_URL`, `LLM_PROVIDER="groq"`, `LLM_API_KEY`, `LLM_MODEL="llama-3.3-70b-versatile"`.
3. `python judge_simulator.py`

## API surface

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/v1/healthz` | Liveness + context counts |
| GET | `/v1/metadata` | Team / model / version |
| POST | `/v1/context` | Push category / merchant / customer / trigger (idempotent by version) |
| POST | `/v1/tick` | Return up to 20 actions for `available_triggers` |
| POST | `/v1/reply` | `send` / `wait` / `end` after merchant/customer message |

## Module map

- `bot.py` — FastAPI app  
- `composer.py` — LLM + validation + `compose()` export  
- `templates.py` — deterministic fallbacks  
- `conversation.py` — reply routing  
- `state.py` — in-memory contexts & cache  
- `llm_provider.py` — Groq + other backends  

## License / challenge

Synthetic dataset for the magicpin challenge only—not for real merchant outreach.
