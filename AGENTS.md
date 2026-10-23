# AGENTS.md

Instructions for coding agents working in this repo. Do not duplicate the README.

## Project

**Zapatería Paso Firme** is a small shoe store (single owner Marisa and one employee) whose star service is a **reservation system**: customers reserve shoes, pay an initial deposit (min **5 €**) and spread the rest over a maximum of **90 days**.

Today everything is manual (notebook + Excel). The repo contains the **store app** (`backend/` + `frontend/`) that replaces the notebook, plus the **n8n** automation layer that must handle everything the app does not do on its own: customer notifications, payment reminders, expirations, cash closures, monthly reports, bank payment emails, Excel migration.

The store app is the **system of record**; n8n is the **automation layer** that talks to it.

Two-way integration:

- **App → n8n**: the app sends `POST /webhook/tienda` with body `{ "evento": "...", "datos": {...} }`. Events: `apartado.creado`, `abono.registrado`, `venta.registrada`, `apartado.vencido`.
- **n8n → app**: the n8n HTTP Request node calls the REST API of the store with header `X-API-Key`, base URL `http://tienda:8000`.

The agent must never give n8n direct access to the database; everything goes through the API, which applies the business rules.

## Stack

The versions that matter are declared in `backend/requirements.txt` and in the `n8n` image tag in `docker-compose.yml`.

- **Backend** (`backend/`): Python 3.12, FastAPI, SQLite. Layers: `routes/`, `controllers/`, `models/`, `schemas/`, `services/`, `middleware/`, tests in `backend/tests/`, seed in `backend/seed.py`.
- **Frontend** (`frontend/`): static HTML/CSS/JS served by the backend. No bundler, no package manager.
- **n8n** (`docker.n8n.io/n8nio/n8n`): workflow engine. Flows are JSON and are managed via the n8n REST API / MCP / visual editor; scheduled flows use the Schedule Trigger.

## Structure

```
backend/            # FastAPI store app (MVC: routes, controllers, models, schemas, services, middleware, tests, seed)
frontend/           # Static store UI (HTML/CSS/JS)
data/               # Sample Excel + bank alert emails (migration & abonos flows)
docs/               # Spanish docs: guia-de-la-app.md, ai-log.md, design-tokens, errores, docker, memoria…
specs/              # One spec per task, written before the n8n flow: `<issue>-<slug>.md`
workflows/          # Exported n8n flow JSONs, versioned in Git/GitHub
scripts/            # Scripts to create demo data via API, export/import n8n flows
docker-compose.yml # tienda + n8n
AGENTS.md
README.md
```

## Workflow

One task = one session = one branch = one PR, and a human reviews before anything is merged.

1. **Spec first.** Write `specs/<issue>-<slug>.md` before the flow. It must include: trigger, inputs, outputs, error cases and acceptance criteria, backed by a domain ontology (entities such as `Cliente`, `Producto`, `Apartado`, `Abono`, `Venta`, `CierreDiario` with attributes, relations and rules).
2. **Agent.** Implement only that task, on that task's branch, following these rules.
3. **PR.** Open it to `main` with the checks green (see "Pull request to `main`").
4. **Human review.** The agent never merges on its own.

Keep `docs/ai-log.md` updated with what the agent was asked, what was accepted or rejected, and where it went wrong.

## AI development with n8n

From the README section **"🤖 Desarrollo Agéntico con IA"**:

**Tools**

- Agent: **OpenCode** (recommended). Alternatives: Aider, Cline, Kilo Code… justify the choice.
- Free models, e.g. OpenCode Zen free, `:free` models on OpenRouter, Groq or Gemini free tier; the same models can be used inside n8n AI nodes.
- **n8n-mcp** (recommended): MCP server that exposes the documentation of all n8n nodes; connected to the API of your instance it lets the agent create, validate and update flows.

**¿En qué ayuda el agente si n8n es visual?**

- **Generate and modify flows:** from a spec, the agent creates the flow JSON and uploads it to n8n (via MCP or REST API). The team opens it in the editor, tests it and adjusts it.
- **Choose and configure nodes:** consult what node to use and its parameters instead of looking it up by hand.
- **Write Code node code** and complex expressions (`{{ $json... }}`).
- **Debug:** pass the error of a failed execution and the flow JSON; propose the fix.
- **Tests:** webhook payloads and scripts to export/import flows.
- **Document:** generate flow docs from the JSON.

The visual editor is still where flows are tested and reviewed; the agent does not replace opening the flow and running it.

**Workflow**

1. **Context before flows:** create/update `AGENTS.md` with the business and its rules, integrated services, n8n flow/node naming conventions, how to import and export flows, and what the agent **must not** do (e.g. write credentials into the JSON, activate flows in production).
2. **SDD & ontologies:** before asking the agent for anything, each automation is defined with Spec-Driven Development in a spec (`specs/`) with the trigger, inputs, outputs, error cases and acceptance criteria, backed by a domain ontology.
3. **Small tasks:** in pairs, split the flows; one Kanban card = one flow (or part of one), one branch and one PR with the exported JSON.
4. **Human review mandatory:** every generated flow is imported, tested with example data and reviewed in the PR before merging. Pay special attention to security: credentials, personal data and exposed webhooks.
5. **Tests as contract:** each flow has test data (pinned data or example payloads) and an expected outcome. If a test fails, fix the flow, not the test.
6. **Traceability:** record relevant prompts and decisions (what was accepted, rejected and why).

## Technical requirements

From the README "Requisitos Técnicos":

1. Store app and n8n with **Docker** (the `docker-compose.yml` is already included).
2. Integration with the store app via its **event webhook** and its **REST API** (HTTP Request node).
3. Google Sheets or similar only for what the app does not store (e.g. cash closure history).
4. At least one scheduled flow (**Schedule Trigger**).
5. Notifications to customers and to the store (Gmail, Telegram, Slack…).
6. Safe **credential** management in n8n, never written in nodes nor uploaded to the repo.
7. Validation of the **business rules** inside the flows (IF / Switch and Code node when needed).
8. **Error handling** (retries, error branches and an **Error Workflow**).
9. Flows exported in **JSON** and versioned in Git and GitHub.
10. Development assisted by **AI agents** with free models.

## Principles

- Follow **DRY** and **SOLID** strictly. For n8n: the store app is the single source of truth — it computes `abonado`, `pendiente`, `precio`, etc.; n8n must use the values returned by the API and must not recalculate them. Business rules are validated inside the flows with IF / Switch / Code nodes.
- If a requirement, task, or design choice is ambiguous or confusing, **stop and ask**. Never decide, invent, or pick an option.

## What the agent must not do

- Never commit secrets or `.env`; those files are local only. Add new variables to `.env.example` with a fake value.
- Never create branches (they come from the board, always off `main`); only `git fetch` and `git checkout`.
- Never merge, and never open a PR, without the human review first.
- Never skip, disable, or delete tests to make the suite pass; fix the cause instead.
- Never invent conventions, dependencies, or scope; if something is missing, stop and ask.
- Never touch files outside the task's scope (e.g. `backend/` when the task is a workflow; `data/` when it is only a flow task).
- Never put credentials in workflow JSON. Never activate flows in production without human review. Never recalculate amounts the app already computes, and never bypass the store API from n8n.

## Branches

Branches are created on GitHub Projects from the task, always off `main`. Locally, only `git fetch` and `git checkout <branch>`. Never create branches.

## Tasks

Board: <https://github.com/orgs/IA-P1-BCN/projects/28>

Work only the current issue. Read its title, body, and acceptance criteria with `gh issue view <number>`. Do not expand scope to sibling subtasks.

## Pull request to `main`

When asked to open a PR to `main`, review first. Do not open the PR until that review is done, and do not "fix" scope on your own.

1. Compare the diff to the issue's acceptance criteria and to these rules (DRY, SOLID, layering, conventions below).
2. Report any gap and ask before continuing.
3. Run the checks for the side that changed:
   - Backend: `ruff check .` and `pytest` (from `backend/`, config in `backend/pytest.ini`).
   - n8n: the exported workflow JSON must be valid and the flow tested (pinned data / example payloads) before merging.
   - Frontend: no automated checks configured (static assets); review manually.
   - Docs: check that referenced files/paths exist.

## Conventions

- Code, names, and comments in **English**. Documentation in **Spanish**.
- Commits: Conventional Commits (`feat(apartados): ...`, `fix: ...`, `test: ...`, `docs: ...`, `refactor: ...`, `chore: ...`).
- Backend dependencies: add them in `backend/requirements.txt`, then `pip install -r backend/requirements.txt`. Never `pip install` a package on its own.
- Frontend: static files; no dependency manager — do not add one without asking.
- Never commit `.env`. If you add a variable, also add it to `.env.example` with a fake value.
- Touch only the side the task requires (`backend/`, `frontend/`, `workflows/`, `specs/`, `docs/`, `data/`).
