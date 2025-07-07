# 📑 PROJECT BLUEPRINT — *Job‑Apply Autonomous Agent*
*(Authoritative technical spec consumed by OpenAI Agent Codex on every task)*  

---

## 1 · Mission Statement  
Deliver an **unattended agent** that:

1. Lands on any supported job‑board URL (Greenhouse → Lever → Workday → LinkedIn Easy Apply).  
2. Navigates to the application form.  
3. Fills every **required** field (text, selects, uploads).  
4. Presses **Submit** and confirms success.

Achieved through a **single‑loop blend** of OpenAI **computer‑use** (pixel‑level planning) and **browser‑use / Playwright** (DOM‑level speed).

“**Done**” for the MVP = the CLI, given a public Greenhouse URL and a PDF resume, exits 0 after saving a “Thank you” confirmation screenshot to `.state/`.

---

## 2 · High‑Level Architecture

```
CLI (apply.py) ─► BlendExecutor ─► Site‑specific FormFiller
                   ├─ pixel path  (pyautogui)
                   └─ DOM  path  (browser‑use / Playwright)
```

* The **BlendExecutor** owns the browser context and decides per action whether to execute via Playwright (fast) or fall back to pixel clicks.  
* **FormFiller** modules are thin adapters: search, open posting, iterate form fields, call the executor.

---

## 3 · Repository Layout

```
/agent            blend executor & helpers
/forms            one module per board (greenhouse.py, lever.py …)
/cli              Typer entry‑points
/tests            pytest suite (fixtures, smoke)
/docs             (optional) extended design notes
/blueprint.md     ← this file (keep in sync)
/AGENTS.md        Codex governing rules
/README.md
```

---

## 4 · Key Classes & Interfaces

| Class | Purpose | Core methods |
|-------|---------|--------------|
| `BlendExecutor` | Owns the loop, browsers & pixel shim | `submit(page, goal_selector:str\|None)->bool` |
| `Scene` | DTO: screenshot + DOM + history | `to_payload()` |
| `FormFiller` (abstract) | Site workflow contract | `search(title)`, `fill(data)`, `submit()` |
| `GreenhouseForm(FormFiller)` | MVP implementation | overrides three hooks |

---

## 5 · Blend Executor — repository details

### 5.1  Upstream reference  
Vendor (`git submodule`) the executor mix‑in from **Upsonic**:

```bash
git submodule add https://github.com/upsonic-ai/upsonic.git vendor/upsonic
cd vendor/upsonic && git checkout v0.60.0   # known‑good tag
```

Copy into our runtime path:

```
/agent/blend_executor.py      (fork of vendor/upsonic/.../blend_executor.py)
/agent/selectors.py           (fork of vendor/upsonic/.../selectors.py)
```

Strip Upsonic‑specific logging so runtime has **zero third‑party deps** outside `requirements`.

### 5.2  Import contract  
Runtime code **never** imports Upsonic.  Only `/agent/blend_executor.py` and helpers live in the package path so:

```python
from agent.blend_executor import BlendExecutor
```

### 5.3  Maintenance  
Once per month:  
```
git -C vendor/upsonic fetch --tags
# cherry‑pick relevant fixes into /agent, bump versions here
```

---

## 6 · Agent Loop – detailed sequence

1. **capture()** → `page.screenshot(type="png")`  
2. **dump()** → `browser_use.dump_tree(trim=True, max_bytes=50_000)`  
3. **fuse()** → `Scene(image_b64, dom_json, history)`  
4. **LLM step** → call `openai.responses.create(model="computer-use-preview", tool="computer_use", scene=payload)`  
5. **selector mapping** → `css = selectors.hit_test(bbox, dom_json)`  
6. **execute** → if `css` hit → `page.click(css)` else pixel click via `pyautogui`.  
7. repeat until confirmation selector disappears.

---

## 7 · Dependency Pinning

| Package | Version | Reason |
|---------|---------|--------|
| `openai-agents` | `^0.9` | orchestration + tracing |
| `browser-use` | `^0.8` | DOM extraction & MCP |
| `playwright` | `^1.44` | browser driver |
| `pyautogui` | `^0.9` | pixel fallback |
| Dev tools | `pytest`, `black`, `typer`, `rich` |

All tested on Python 3.11 **and 3.12**.

---

## 8 · API Cheat Sheets

### 8.1  OpenAI `responses.create`
| Field | Example |
|-------|---------|
| `model` | `"computer-use-preview"` |
| `tool` | `"computer_use"` |
| `scene` | base‑64 PNG + trimmed DOM JSON + history |
| Return | one of four actions: `click`, `type`, `scroll`, `screenshot` |

### 8.2  `browser_use`
```python
from browser_use import Browser
browser = Browser(headless=True)
page = await browser.new_page()
dom = await page.dump_tree(trim=True)
await page.exec_click(css="#submit")
```

---

## 9 · Autonomy Strategy

1. **Recursive tasks** — Codex spawns child tasks if >30 min work.  
2. **Token thrift** — DOM snapshot ≤ 50 kB.  
3. **Self‑healing** — after 3 pixel mis‑clicks → reload page & reset loop.  
4. **State checkpoint** — save `scene` + action JSON to `.state/`.

---

## 10 · Testing Plan

| Level | Fixture | Assertion |
|-------|---------|-----------|
| Unit | dummy HTML in `tests/fixtures/` | `BlendExecutor.submit()` returns `True` |
| Integration | live Greenhouse sandbox | confirmation selector found |
| Regression | nightly replay of last N jobs | DOM diff + screenshot hash |

---

## 11 · Observability

* **Tracing** → Agents‑SDK decorator (`OPENAI_AGENTS_DISABLE_TRACING=1` to opt‑out).  
* **Logs** → ND‑JSON per step in `logs/<date>.ndjson`.  
* **Metrics** → success‑rate, steps/job, tokens/job.

---

## 12 · Deployment / Runtime

```bash
poetry install
playwright install chromium --with-deps
export OPENAI_API_KEY=…
export GH_TOKEN=…
python cli/apply.py --resume ./cv.pdf https://boards.greenhouse.io/acme/jobs/12345
```

**Dockerfile (excerpt)**  
```Dockerfile
FROM python:3.12-slim
ENV PIP_NO_CACHE_DIR=1
COPY . /app
WORKDIR /app
RUN poetry install && playwright install chromium --with-deps
ENTRYPOINT ["python","cli/apply.py"]
```

---

## 13 · Future Extensions

* Captcha‑solver plugin via anti‑captcha API.  
* OAuth/2FA hand‑off to human approval agent.  
* Vector feedback loop for failure fine‑tuning.

---

> **This blueprint is authoritative.**  
> Any architectural change must update this document **and** the tests in the same PR.
