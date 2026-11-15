# ZIGURAT-MBIMPT-M7T2

BIM data visualisation dashboard with an AI assistant, built with Streamlit.
It loads IFC models, extracts quantities and properties, and lets the user
create charts and filters manually or by chatting with an LLM (Claude, OpenAI,
Gemini or any OpenAI-compatible endpoint).

ZIGURAT Institute of Technology, MBIMPT module 7, task 2. The original code
was provided by the professor, Ítalo Guedes. This file records every change
made to it since.

## Running

Locally:

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
streamlit run main.py
```

Streamlit Community Cloud: point the app at `main.py` in this repository.
Optionally set `LLM_API_KEY` under *Settings > Secrets*; otherwise paste the
key in the sidebar.

GitHub Codespaces: opening the repository in a codespace installs the
requirements and starts the app on port 8501 (see `.devcontainer/`).

## Changelog

Baseline: the professor's original `main.py` and `requirements.txt`, commit
`cbbe97e` (2026-11-15).

### 2026-11-15: Default Gemini model changed to 3.6 Flash (`03ffca0`)

Problem: with Gemini selected, chat requests timed out even after the
timeout was raised to 180 s.

Diagnosis, measured against the live Gemini API (REST `generateContent`,
the app's real system prompt plus a sample data context):

| Test | Model | Time | Thinking tokens |
|---|---|---|---|
| App request, 5 runs | `gemini-3.8-flash` | 107 s, 29 s, 45 s, 44 s, 58 s | 700 to 980 |
| App request, 4 runs | `gemini-3.6-flash` | 5.0 to 7.7 s | 770 to 1,240 |
| One-line prompt, streamed | `gemini-3.8-flash` | 24.5 s before first byte | 123 |
| One-line prompt, streamed | `gemini-3.6-flash` | 2.5 s | 324 |
| One-line prompt, streamed | `gemini-3.5-flash-lite` | 1.7 s | none |
| App request with low thinking | `gemini-3.8-flash` | 54 s | not reported |

- Both models think about the same amount, so the thinking volume is not
  the cause.
- `gemini-3.8-flash` waits about 24 s on Google's side before it sends
  anything, even for a trivial prompt, and its total time varies widely.
  At busy times it passes the 180 s client timeout. Lowering the thinking
  level did not fix it.
- Gemini developer forum threads from October and November 2026 report the
  same 3.8 Flash latency, 503 errors and read timeouts.

Changes in `main.py`:

- `PROVIDERS["Google Gemini"]["model"]`: `gemini-3.8-flash` to
  `gemini-3.6-flash`. The same request returns valid JSON in about 5 s.
- Removed `"temperature": 0.2` from the Gemini `generationConfig`. Gemini
  3.6 and later ignore sampling parameters (`temperature`, `top_p`,
  `top_k`), and Google says future models will reject them with HTTP 400.

Also added in this round:

- `.gitignore` for Python bytecode (`__pycache__/`), virtual environments,
  local secrets (`.streamlit/secrets.toml`, `.env`) and OS/editor files.
  The repository had no `.gitignore`.
- This README, replacing the one-line placeholder.

### 2026-11-15: Gemini model and timeout updated (`babb1cd`)

- Gemini default model: `gemini-2.5-flash` to `gemini-3.8-flash`.
- `_post()` HTTP timeout: 90 s to 180 s.
- Result: Gemini chat requests started timing out (see the entry above).

### 2026-11-15: Dev container added (`7d33fd2`)

- `.devcontainer/devcontainer.json` for GitHub Codespaces: Python 3.11
  image, installs `requirements.txt`, and starts
  `streamlit run main.py` on port 8501.

## Possible further improvements (not implemented)

- Retry the Gemini call once on HTTP 503 or a timeout. The forum reports
  show 3.8 Flash capacity errors, and other models can have the same issue.
- `call_llm()` reads only the first response part
  (`parts[0]["text"]`). Joining the text of all parts would be safer if a
  model splits its answer.
- The module docstring tells users to run `app.py`, but the file is
  `main.py`. Streamlit Cloud must point at `main.py`.
- `requirements.txt` uses minimum versions (`>=`). Pinning exact versions
  would make Streamlit Cloud builds reproducible.
- The sidebar hint for Gemini keys is `AIza...`. Google AI Studio keys do
  not always use that prefix (the key used for testing started with `AQ.`),
  so the hint can confuse users.
- The dev container starts Streamlit with CORS and XSRF protection
  disabled, which is the Codespaces template default. Keep those flags
  for the Codespaces preview only, not for a public deployment.
