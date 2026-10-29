# pov-init-agent — get a head start on your LangSmith trial

A new LangSmith workspace starts empty, which makes it hard to know where to begin.
This repo fixes that. It ships a small but real AI agent and **one command** that
connects to your workspace with an API key and pre-populates it with everything you'd
build over the first few weeks of a proof-of-concept:

- **Traces** — multi-step runs from a working agent
- **A dataset** — golden examples for evaluation
- **An evaluator, offline _and_ online** — the same quality check run on test data and on live traffic
- **An annotation queue** — a human-review workflow with a scoring rubric
- **Automations** — rules that route live traffic to the queue and grow your dataset
- **Context Hub** — the agent's prompt, memory, and a skill, managed in LangSmith

You get a workspace that already looks like a project in flight, and this README
doubles as a guided tour of each LangSmith concept.

The agent is a **research report writer**: give it a topic, it plans an outline, drafts
each section, and returns a structured markdown report. It needs **only an LLM API key** —
no web search or other external services — so it's easy to run.

---

## Prerequisites

1. **A LangSmith account + API key.** Create a key under **Settings → API Keys**.
   - Use a **workspace-scoped** key (the default). It resolves its own workspace and
     needs nothing else.
   - If you only have an **org-scoped** key, also set `LANGSMITH_WORKSPACE_ID` in `.env`.
2. **One LLM provider key** — Anthropic, OpenAI, or Google. You only need the one your
   `MODEL` points at.
3. **[uv](https://docs.astral.sh/uv/)** for Python (`curl -LsSf https://astral.sh/uv/install.sh | sh`).

---

## Quickstart

```bash
git clone <this-repo> && cd pov-init-agent
uv sync                       # create the venv and install everything
cp .env.example .env          # then open .env and fill in your keys
uv run pov-bootstrap          # populate your workspace (~a few minutes)
```

`pov-bootstrap` prints its progress step by step and finishes with a link to your
workspace. Open it and start exploring — the guided tour below tells you what to look at.

> The bootstrap is **idempotent**: re-running it won't create duplicates. Re-run it
> any time you change the prompt, the dataset, or the seed topics.

---

## Guided tour: what got created, and what each thing is

### 1. Tracing — see what your agent actually did
Open the **Tracing** project named **`pov-report-writer`**. Each seed topic produced a
trace: a tree of steps showing the agent planning, drafting sections, and synthesizing
the final report — with inputs, outputs, token counts, and latency at every step.
Tracing is the foundation everything else builds on.
→ [Observability docs](https://docs.langchain.com/langsmith/observability)

### 2. Datasets & Experiments — measure quality on known-good examples
Open **Datasets & Experiments → `pov-report-writer-eval`**. It holds a dozen report
topics, each with the points a good report *must cover*. Running the agent over this
dataset produces an **experiment** with a score per example — that's **offline
evaluation**: testing against fixed examples before you ship.
→ [Evaluation concepts](https://docs.langchain.com/langsmith/evaluation-concepts)

### 3. Evaluators — the same check, offline and online
This repo defines one quality metric and deploys it **both** ways:
- **Offline** (`report_structure` + an LLM-as-judge `coverage` score) runs in the
  experiment above, against your golden dataset.
- **Online** (`report-structure`) is attached to the live tracing project and scores
  **every production trace** automatically. Find it under the project's **Evaluators**
  tab.

Offline tells you "is the new version better than the old one on our test set?" Online
tells you "how is quality trending on real traffic right now?"
→ [Online evaluators](https://docs.langchain.com/langsmith/code-evaluator-ui) ·
  [Offline evaluation](https://docs.langchain.com/langsmith/evaluate-llm-application)

### 4. Annotation queues — put a human in the loop
Open **Annotation Queues → `pov-report-review`**. A handful of generated reports are
queued for human review, with a **rubric**: an overall quality score, a pass/fail
"factually grounded" check, and a free-text notes field. This is how subject-matter
experts grade outputs and build trustworthy datasets.
→ [Annotation queues](https://docs.langchain.com/langsmith/annotation-queues)

### 5. Automations — make the workspace work on its own
Back in the **`pov-report-writer`** project, open **Automations**. Two rules are live:
- **Route sampled reports to the review queue** — 50% of new runs go to humans.
- **Grow the eval dataset from production** — 30% of new runs are added to a dataset,
  so your test set reflects real usage over time.

Rules match traces with a filter and fire an action (queue, dataset, webhook, or an
evaluator). They're how a trial graduates into a real feedback loop.
→ [Automation rules](https://docs.langchain.com/langsmith/rules)

### 6. Context Hub — manage the agent's brain in LangSmith
Open **Context Hub**. You'll see an **agent repo** `pov-report-writer` containing the
agent's **system prompt**, its long-term memory **`AGENTS.md`**, and a link to the
**`report-style-guide`** skill repo. Editing these here changes the agent's behavior
**without a redeploy** — and the agent writes lessons back to `AGENTS.md` as it runs.
→ [Context Hub](https://docs.langchain.com/langsmith/context-hub)

---

## The agent, explained

It's a [**Deep Agent**](https://docs.langchain.com/oss/python/deepagents/overview)
(LangGraph under the hood) wired to a **composite filesystem backend**:

```
CompositeBackend(
    default = ContextHubBackend("pov-report-writer"),   # prompt, /AGENTS.md, /skills/ live in LangSmith
    routes  = { "/scratch/", "/large_tool_results/", "/conversation_history/" -> StateBackend() },
)
```

- **Durable files** (the prompt, the `AGENTS.md` memory, and the skill) are served from
  the **Context Hub**, so they're versioned and editable in LangSmith.
- **Ephemeral files** (section drafts the agent writes to `/scratch/`, plus the deep
  agent's own internal scratch paths) go to an in-memory `StateBackend`, so they never
  clutter the Hub with commits.

At runtime the agent: **plans** an outline → **drafts** each section to `/scratch/` →
**synthesizes** the final report. That's what produces the rich, multi-step traces.

The graph is defined in `src/pov_agent/agent.py` and registered in `langgraph.json`.

---

## Run and modify the agent yourself

Launch it locally in LangGraph Studio (hot-reloads on edits):

```bash
uv run langgraph dev
```

Then send it a topic and watch the trace appear in LangSmith.

**Customize:**
- **Prompt / memory / skill** — edit them in the Context Hub (or edit
  `src/pov_agent/langsmith/hub_content/` and re-run `uv run pov-bootstrap` to push).
- **Dataset topics** — edit `src/pov_agent/langsmith/seed_data.py`.
- **Evaluators** — `src/pov_agent/langsmith/evaluators.py` (offline) and
  `online_eval.py` (online). The structure rule is intentionally duplicated between
  them because the online evaluator runs in a sandbox that can't import the repo.
- **Object names** — all in `src/pov_agent/config.py`.

---

## Deploy the agent (optional)

Run it locally, or ship it to **LangSmith Deployments** for a hosted, always-on endpoint:

```bash
uv run langgraph dev      # local dev server + Studio
uv run langgraph deploy   # deploy to LangSmith Deployments (requires Docker)
```

Set `MODEL` and your provider key (e.g. `ANTHROPIC_API_KEY`) as env vars on the
deployment — `LANGSMITH_API_KEY` is injected automatically. Run `uv run pov-bootstrap` in
the same workspace first so the Context Hub repo exists. The deployment spends whatever
provider key you set on it.

→ [LangSmith Deployments docs](https://docs.langchain.com/langsmith/deploy-to-cloud)

---

## Choosing a model (provider-agnostic)

The agent and the LLM-as-judge both read the `MODEL` env var, in `provider:model` form:

```bash
MODEL=anthropic:claude-sonnet-4-6     # default
# MODEL=openai:gpt-5.1
# MODEL=google_genai:gemini-2.5-flash
```

Set the API key for whichever provider you pick (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`,
or `GOOGLE_API_KEY`). Nothing else changes.

> **Optional — LLM Gateway.** For centralized key management and spend controls, you can
> route model calls through the [LangSmith LLM Gateway](https://docs.langchain.com/langsmith/llm-gateway)
> instead of using a provider key directly. Not required for this repo.

---

## Project layout

```
src/pov_agent/
├── config.py                 # model + all LangSmith object names
├── agent.py                  # the deep agent (composite backend + Context Hub)
└── langsmith/
    ├── bootstrap.py          # `pov-bootstrap` — orchestrates every step
    ├── context_hub.py        # pushes prompt + AGENTS.md + skill to the Hub
    ├── datasets.py           # creates the eval dataset
    ├── traces.py             # runs the agent to generate seed traces
    ├── annotation_queues.py  # feedback configs + queue + seeds runs
    ├── evaluators.py         # offline: structure (code) + coverage (LLM judge)
    ├── online_eval.py        # online: self-contained structure evaluator
    ├── offline_eval.py       # runs the offline experiment
    ├── automations.py        # creates the run rules (REST)
    ├── seed_data.py          # the report topics + must-cover points
    └── hub_content/          # the prompt, AGENTS.md, and SKILL.md pushed to the Hub
```

---

## Notes

- Secrets live in `.env` (gitignored). Only `.env.example` is committed. Never commit a
  real key.
- A note on naming: LangSmith and Deep Agents use **`AGENTS.md`** for an agent's memory
  file; that's the "AGENT.md" this repo creates.
