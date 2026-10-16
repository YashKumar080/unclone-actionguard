# ActionGuard Lab — Project Context

> **Purpose:** This file is the shared source of truth for all three developers and any AI coding assistant working on ActionGuard Lab. Read it before changing code.
>
> **Project scope:** UNCLONE hackathon, Track C (Agents). Build a working security/policy gateway for a **simulated** AI voice agent and sensitive banking actions. This is a demo, not a real banking system.

## 1. Problem and goal

Voice agents can be attacked with synthetic speech, cloned voices, replayed audio, or a genuine but unauthorized speaker. A voice match or deepfake detector alone is not enough to safely authorize sensitive actions.

Build **ActionGuard Lab**, which:
1. accepts a voice recording for a simulated banking request;
2. sends audio to DetectifAI for deepfake detection when configured;
3. uses the detector's documented verdict plus speaker-verification information only if the speaker API has been implemented from its exact documentation;
4. applies an explicit action-risk policy;
5. returns a decision such as `ALLOW`, `STEP_UP`, `HUMAN_REVIEW`, or `BLOCK`;
6. records a decision/audit event; and
7. provides an Attack Lab for comparing policy outcomes on a small, consented set of test clips.

**No real banking transactions are performed.** Use a mock bank state only.

## 2. Non-negotiable safety and demo rules

- Use only audio the speaker has consented to share and that the team is authorized to use.
- Do not clone real people without consent. Do not use participant, judge, or staff voices unless organizers explicitly permit it.
- The demo must be accessible to judges without requiring them to sign in.
- Do not fabricate API responses, detection scores, speaker matches, latency, test results, or performance claims.
- If DetectifAI is unconfigured or fails, show a clear error. Never replace a failed live result with a made-up score or silently label it as live.
- If prerecorded results are ever shown, label them clearly as **recorded/demo data**, not live detection.
- Audio uploaded to DetectifAI is transmitted to that provider. Do not claim that such audio stays only on the local machine.
- Keep API credentials secret. Never print, log, return to the browser, commit, or place the DetectifAI API key in frontend code.
- This project is a prototype and its small hand-made test set is **not** a benchmark or proof of real-world security.
- Disclose AI coding tools used, as required by the event.
- Do not claim that a detector's verdict proves a speaker's identity or proves fraud.

## 3. Technology and environment

Target environment: Windows, Python 3.10 or newer.

Backend:
- FastAPI
- Uvicorn
- `requests`
- `python-dotenv`
- `imageio-ffmpeg`
- `pytest`
- `httpx`
- `edge-tts` may be used for clearly labeled, non-impersonating synthetic test speech.

Frontend:
- Plain HTML, CSS, and JavaScript.
- Do not introduce React or another frontend framework unless the team explicitly agrees first.

Storage:
- In-memory mock banking state for the demo.
- JSON Lines (`.jsonl`) audit log if needed.
- No database is required for the first working version.

## 4. DetectifAI integration contract

Base URL: `https://platform.detectif.ai/v1`

Documented endpoint:
- `POST /v1/deepfake/detect`
- Send a multipart form upload using field name `file`.
- Authentication: `Authorization: Bearer <DETECTIF_API_KEY>`.
- Use an `Idempotency-Key` header for safe retries where appropriate.
- Quickstart requirements: audio longer than 4 seconds and no longer than 120 seconds.

The response includes fields such as:
- `request_id`
- `operation_id`
- `result.verdict`: `real`, `fake`, or `not_sure`
- `label`
- legacy `is_synthetic`
- `score` from 0 to 1, where a higher score indicates a greater likelihood of AI-generated audio
- `threshold`
- `duration_seconds`

Implementation rules:
- Prefer `result.verdict` as the primary detector result. Preserve `not_sure` as uncertainty; do not convert it into `real`.
- A numeric score is not proof. Do not invent or add a numeric score cutoff to the policy.
- Only `app/detectif_client.py` should know the DetectifAI HTTP endpoint, authentication, and response parsing details.
- Read the API key from the server-side environment variable `DETECTIF_API_KEY`, typically loaded from `.env`.
- Never expose the API key to the frontend or in API responses/logs.
- Show the real provider response fields needed for the demo and keep the raw response server-side where appropriate.
- If the key is missing, return a clear configuration error (suggested HTTP status `503`).
- If the provider fails or returns an unusable response, return a clear upstream/service error (suggested HTTP status `502`). Do not make a policy decision as though detection succeeded.
- Use appropriate timeouts and safe error handling. Do not log the secret or sensitive audio bytes.
- Validate and convert audio only as needed. A 16 kHz sample rate is a project placeholder, **not** a vendor-confirmed requirement; do not treat it as one without evidence.
- Run a real API smoke test only when the team has a valid key and consented audio. Record whether it succeeded; do not claim success until confirmed.

### Speaker verification limitation

DetectifAI documentation reviewed so far identifies speaker enroll/verify/list/delete capabilities, but the exact endpoint-specific request and response schemas have not yet been confirmed from the supplied reference. **Do not invent endpoint paths, request fields, response fields, or speaker-match results.** Before implementing speaker enrollment or verification, consult the exact current documentation and update this file with verified schemas. Until then, the app must represent speaker verification as unavailable/unconfigured rather than fabricate `MATCH` or `NO_MATCH`.

### Suggested adapter interface

A proposed internal interface is:

```python
analyze(
    wav_path,
    speaker_reference_id=None,
    idempotency_key=None,
)
```

Return a normalized result containing documented detector data such as `verdict`, `score`, `request_id`, `latency_ms`, and `raw`. Include speaker-match fields only after the documented speaker API has been implemented. Do not invent values. `latency_ms` should be measured by the application if displayed, not hardcoded.

## 5. Shared decision vocabulary

Detector verdict:
- `real`: provider classified the audio as real.
- `fake`: provider classified the audio as fake/synthetic.
- `not_sure`: provider could not reliably classify it.

For policy rules, normalize detector outcomes to:
- `CLEAN` when verdict is `real`;
- `SYNTHETIC` when verdict is `fake`;
- `UNCERTAIN` when verdict is `not_sure`.

Speaker verification, only after a documented and working integration:
- `MATCH`
- `NO_MATCH`
- otherwise unavailable/unknown.

Final gateway decisions:
- `ALLOW`
- `STEP_UP`
- `HUMAN_REVIEW`
- `BLOCK`

`STEP_UP` means require an additional verification step; it is not itself proof of fraud. `HUMAN_REVIEW` means do not perform the requested simulated action until a human reviews it. `BLOCK` means reject the simulated request.

## 6. Required action-risk policy

Implement these rules exactly, in this order. Keep policy logic separate from HTTP routes and UI code.

### LOW risk — `get_branch_info`

- Always return `ALLOW`.
- If the detector result is not `CLEAN`, set an audit flag for the unusual audio.
- This low-risk action does not require speaker verification.

### MEDIUM risk — `check_balance`

1. `SYNTHETIC` → `HUMAN_REVIEW`
2. `CLEAN` + `MATCH` → `ALLOW`
3. Any other combination → `STEP_UP`

If speaker verification is unavailable, do not pretend it matched. The result should follow the “any other combination” rule and require `STEP_UP` for a clean detector result.

### HIGH risk — `change_phone_number`, `add_beneficiary`, `transfer_funds`

Apply the first matching rule:

1. `SYNTHETIC` + `NO_MATCH` → `BLOCK`
2. `SYNTHETIC` otherwise → `HUMAN_REVIEW`
3. `NO_MATCH` with `CLEAN` or `UNCERTAIN` → `HUMAN_REVIEW`
4. Otherwise → `STEP_UP`

**High-risk actions must never be `ALLOW` on voice alone.** If speaker verification is unavailable, treat it as unknown—not as `MATCH` or `NO_MATCH`—and follow the otherwise/step-up branch unless another rule clearly applies.

## 7. Baselines for comparison only

Attack Lab may compare the policy against these deliberately simple baselines:

- **Detector-only baseline:** `BLOCK` if verdict is `fake`; otherwise `ALLOW`, including `not_sure`.
- **Voiceprint-only baseline:** `ALLOW` if speaker result is `MATCH`; otherwise `STEP_UP`.

These are comparison baselines, not the ActionGuard policy. Label them clearly in the UI and reports. Do not present them as recommended production security policies.

## 8. Attack Lab

Purpose: demonstrate how outcomes differ across a small, consented set of test recordings. It is an illustrative demo, not a scientifically representative benchmark.

Use these scenario labels:
- `genuine_customer`
- `genuine_other`
- `tts_stranger`
- `clone_customer`
- `replay_customer`

Use only authorized, consented team recordings. Obtain explicit organizer permission before using participant audio. Never imply that an untested scenario was tested.

Compare:
- ActionGuard policy
- detector-only baseline
- voiceprint-only baseline, only if speaker verification is truly implemented

Useful counts/metrics:
- attacks approved
- attacks held for review
- attacks stepped up
- genuine requests escalated or blocked (friction)

Show the number of clips behind each result. For a small sample, prefer counts and transparent per-clip outcomes over sweeping percentages or claims. If a detector/API call fails, mark that case as an error and exclude it from successful detection comparisons; do not silently count it as a real or fake result.

## 9. Suggested project structure

```text
unclone-actionguard/
├── PROJECT_CONTEXT.md
├── README.md
├── .gitignore
├── .env.example
├── requirements.txt
├── app/
│   ├── __init__.py
│   ├── config.py
│   ├── main.py
│   ├── audio_utils.py
│   ├── detectif_client.py
│   ├── policy.py
│   ├── tools.py
│   ├── agent.py
│   ├── gateway.py
│   └── lab.py
├── static/
│   ├── index.html
│   ├── app.js
│   └── style.css
├── tests/
├── scripts/
├── docs/
├── recordings/
├── attack_clips/
└── data/
```

This is a suggested layout; small changes are okay if the team agrees and the README stays accurate.

## 10. File ownership and collaboration

To reduce merge conflicts, agree on ownership before parallel work:

- **Developer 1 — Lead/API:** project setup, configuration, DetectifAI adapter, API smoke test, integration and final merge.
- **Developer 2 — Policy/Attack Lab:** `app/policy.py`, policy tests, gateway policy flow, Attack Lab metrics/comparison logic.
- **Developer 3 — UI/Data/Story:** `static/index.html`, `static/app.js`, `static/style.css`, consented test clips, demo flow, README/presentation evidence.

Shared contracts (`PROJECT_CONTEXT.md`, normalized result shape, route names, response schema) must be agreed before parallel implementation. Do not silently change a shared contract; discuss it and update this file.

## 11. Secrets, ignored files, and environment setup

- Commit `.env.example`, not `.env`.
- Never commit API keys, recordings containing personal data, or private attack clips.
- Add `.env`, `recordings/`, `attack_clips/`, and sensitive generated `data/` files to `.gitignore` as appropriate.
- Keep only consented, approved demo assets in the repository. If an asset must remain private, keep it out of Git and document how the team can obtain it securely.

Example `.env.example`:

```dotenv
DETECTIF_API_KEY=
```

Typical Windows setup in PowerShell, from the project root:

```powershell
py -3 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
Copy-Item .env.example .env
```

If PowerShell blocks activation, use this for the current terminal session only, if permitted:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\.venv\Scripts\Activate.ps1
```

Run the app (adjust module path if the agreed project entry point differs):

```powershell
uvicorn app.main:app --reload
```

Run tests:

```powershell
pytest -q
```

Do not commit the `.venv` directory.

## 12. API and implementation expectations

- Keep routes thin: validate input, call service/policy functions, return structured results.
- Keep `app/policy.py` deterministic and testable without calling an external API.
- Use a clear, documented normalized result schema shared by the backend and frontend.
- Do not mutate mock bank state unless the policy has explicitly authorized the simulated action; for high-risk actions, voice alone never authorizes execution.
- Do not execute a simulated action after a detection/API error.
- Keep audit records useful but minimal; avoid secrets and unnecessary personal/audio data.
- Add tests for each policy branch, error handling, and invalid uploads.
- Add upload validation and sensible file size/type limits. Enforce the provider's documented audio-duration constraints before calling it where feasible.
- The frontend must distinguish live results, errors, and recorded/demo examples.
- Use clear status labels and explain uncertainty in plain language.
- Avoid claims such as “100% accurate,” “fraud-proof,” or “production-ready.”

## 13. Minimum acceptance checklist

### Setup and collaboration
- [ ] `PROJECT_CONTEXT.md` is at the repository root and committed.
- [ ] `.env` and private recordings are ignored by Git.
- [ ] All three developers can clone the repo and run the app/tests.

### DetectifAI
- [ ] API key is read server-side from `DETECTIF_API_KEY`.
- [ ] API key is never returned to the client or written to logs.
- [ ] Real detector verdict is used; `not_sure` remains uncertain.
- [ ] Missing key/provider failure is shown as an error, not a fake result.
- [ ] A real API smoke test is marked successful only after it actually succeeds.
- [ ] Speaker verification is not claimed until its exact schema and integration are verified.

### Policy and gateway
- [ ] Every LOW, MEDIUM, and HIGH rule is covered by automated tests.
- [ ] High-risk actions cannot be allowed by voice alone.
- [ ] No action executes after a detector/API error.
- [ ] Audit records the decision and relevant flags without secrets.

### Attack Lab and demo
- [ ] Only consented and authorized clips are used.
- [ ] Baselines are clearly distinguished from the ActionGuard policy.
- [ ] Counts show the actual sample size; failed API calls are shown as errors.
- [ ] Demo works for judges without login.
- [ ] Recorded results, if any, are labeled as recorded—not live.
- [ ] Presentation contains exactly five slides in this order: **Problem, Solution, DetectifAI, Impact, Scale**.
- [ ] AI coding tools used are disclosed as required by the event.

## 14. Build order and gates

1. **Foundation:** create repo, this file, `.gitignore`, environment example, minimal FastAPI app, and a passing test.
2. **Gate G1 — Shared contract:** agree route names, normalized detector result, policy inputs/outputs, and error schema before parallel integration.
3. **DetectifAI adapter:** implement the documented deepfake endpoint and verify it with a real, authorized smoke test when credentials are available.
4. **Policy:** implement the deterministic rules and tests independently of the provider.
5. **Gateway integration:** connect upload → validation → detection → policy → audit; do not execute mock actions on errors.
6. **UI:** show upload, requested action, real detector result, policy decision, reasons, and errors clearly.
7. **Attack Lab:** run authorized clips, show per-clip outcomes and baseline comparisons, and state the sample size.
8. **Final verification:** clean setup, tests, judge-accessible demo, no secrets in Git, and five-slide deck in the required order.

If any requirement conflicts with the hackathon's latest official rules, follow the official rules and update this file transparently.
