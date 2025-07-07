# AGENTS.md — Global Guidance for OpenAI Agent Codex
*(Parsed at the beginning of **every** Codex task; treat as an immutable prompt unless explicitly updated in the same PR.)*

---

## 0‑A  Autonomous Kick‑off

Upon first read Codex **must**:

1. Decompose the TODO list (§7) into granular child tasks.  
2. Queue and execute those tasks automatically.  
3. Iterate until the MVP “Done” criteria (§10) are met — **without further human prompts**.

> If any task would require secrets not yet present, create a placeholder env‑var and open a PR that documents the need.

---

## 0‑B  Context Pointer

*Architecture, data‑flow diagrams, and API cheat‑sheets live in **`blueprint.md`**.*  
Refer to it whenever technical ambiguity arises.  Any change to architecture requires a simultaneous update to `blueprint.md` **and** accompanying tests.

---

## 1  Coding Conventions

| Category    | Requirement |
|-------------|-------------|
| Language    | Python **3.11 or 3.12** only. |
| Formatter   | `black` **line‑length 120**. |
| Typing      | `from __future__ import annotations` & full type‑hints on public APIs. |
| Lint Gate   | `pytest -q` **and** `black --check` must pass before each commit. |
| Pylint      | Run `pylint` with rules: `unused-import, unused-variable`; score not enforced, but **0 errors**. |
| Imports     | Absolute, alphabetised, one per line. |
| Docs        | Public functions/classes need Google‑style docstrings. |

Codex **must** run these gates automatically and refuse to commit if they fail.

---

## 2  Project Layout (enforced)

```
/agent            blend executor & helpers
/forms            per‑board workflows (greenhouse.py, lever.py …)
/cli              Typer entry‑points
/tests            pytest suite (fixtures, smoke)
/vendor/upsonic   git submodule (reference executor)
/docs (optional)  extended notes
/blueprint.md     authoritative spec
/AGENTS.md        this file
/README.md
```

Creating or modifying files outside these paths **requires** updating `blueprint.md` in the same PR.

---

## 3  Dependencies & Runtime

Managed with **Poetry** (`pyproject.toml`, semantic‑version pins):

| Package            | Version |
|--------------------|---------|
| `openai-agents`    | `^0.9` |
| `browser-use`      | `^0.8` |
| `playwright`       | `^1.44` |
| `pyautogui`        | `^0.9` |
| Dev tools          | `pytest`, `black`, `typer`, `rich` |

> **Third‑party code:**  
> The *only* vendored dependency is the **Upsonic executor mix‑in** (added as git submodule under `/vendor/upsonic`). Runtime code **must not** import Upsonic directly; instead, copy `blend_executor.py` and `selectors.py` into `/agent` and strip Upsonic‑specific logging.

---

## 4  Security & Secrets

* Never hard‑code credentials. Read from environment variables documented in `blueprint.md`.  
* Screenshots containing PII are written only to `.state/` and rotated daily (max 30 days).  
* If a task needs a new secret, create a placeholder env‑var and update both docs in the same PR.

---

## 5  Autonomy Strategy

1. **Recursive decomposition** – break tasks >30 min into children automatically.  
2. **Single‑loop principle** – all navigation uses `BlendExecutor` (pixel + DOM).  
3. **Token thrift** – DOM snapshot fed to computer‑use ≤ 50 kB.  
4. **Self‑healing** – after 3 pixel mis‑clicks reload page & restart loop.  
5. **State checkpoint** – each step writes scene + action JSON to `.state/`.

---

## 6  Codex Task Template

```
TASK: <imperative, single responsibility>
FILES TO EDIT: <paths>
ACCEPTANCE: <tests / lint rules>
CONTEXT: <pointer to blueprint.md sections>
```

Codex must follow this template when spawning child tasks.

---

## 7  TODO List (MVP)

1. **Scaffold repository & Poetry config** matching §2 layout.  
2. **Add Upsonic submodule** under `/vendor/upsonic` @ `v0.60.0`; copy `blend_executor.py` & `selectors.py` into `/agent` and strip logging.  
3. **Implement Scene helper** in `/agent/scene.py` and wire into `BlendExecutor`.  
4. **Create `forms/greenhouse.py`** that can open any public Greenhouse URL and walk to the first application page.  
5. **Smoke test**: headless HTML fixture verifying `BlendExecutor.submit()` returns `True`.  
6. **CLI wrapper** `cli/apply.py` exposing `apply --resume <file> <url …>`.

---

## 8  Validation Gates

* **Unit tests** – ≥80 % diff coverage.  
* **Smoke test** – live headless Greenhouse submission fixture.  
* **Static analysis** – `black --check`, `pytest -q`, and pylint rules.  
* **Docs coherence** – modify `blueprint.md` & this file if architecture changes.

---

## 9  Prohibited Actions

* Bypassing formatter, linter, or tests.  
* Introducing new external services without env‑var placeholders & doc updates.  
* Editing `README.md`, `blueprint.md`, or this file without updating tests.

---

## 10  Done Definition (MVP)

`cli/apply.py`, given a **public Greenhouse URL** and a PDF resume, completes the form headlessly, exits **0**, and stores a “Thank you” screenshot in `.state/`.

All gates in §8 pass.

---

> **Follow this AGENTS.md verbatim.**  Any ambiguity → consult `blueprint.md`.  
> Non‑MVP requests should be marked with `# backlog:` comments for future tasks.
