---
status: in-progress
---
## Description

A web app that builds a personalised tech-career roadmap from a person's CV and preferences. They upload a CV (PDF or DOCX), pick a target track, seniority and company tier, and answer five quick questions (location, work mode, weekly hours, budget, learning style). A pipeline of four LLM agents profiles the CV, scouts live job listings, works out the skill gap and its timeline, and returns phased milestones (courses, projects, events) with advice on when and how to update the CV. The roadmap is grounded in real job data, not generic advice.

Stack from the person's plan: Python, FastAPI and Pydantic AI for the backend; `pymupdf` and `python-docx` for CV parsing; a job-search API and a web-search API (not scrapers) for live data; Next.js, React and Tailwind for the frontend. Users bring their own LLM API key.

Depends on no other ticket.

## Agent instructions

1. Read `AGENTS.md`, and the Pydantic AI docs for structured agent output and parallel runs. Check how the library behaves rather than assuming.
2. Propose each of these decisions to the person, and wait for their sign-off before writing any code:
   - The LLM provider, and how a user supplies their own API key. The key must never be stored, logged or committed, and never pass through the chat.
   - The job-search API (SerpApi Google Jobs, Adzuna or Jooble) and the web-search API (Tavily or Serper), and whose keys they use.
   - Pydantic AI or LangGraph for orchestration.
   - The names of the backend and frontend folders.
   - The test runner and linter for each side.
   - The gap scoring rules. The plan gives points per missing skill, paradigm and level jump, and divides by weekly hours, but it doesn't say which score maps to 6 months, 1 year or 2 years. Its hour bands (under 5, 5-10, 10-20, 20+) also don't line up with its thresholds (under 7, 15 or more). Propose exact thresholds and a precedence order.
   - The 100-word minimum for an extractable CV, and what counts as "no recognisable skills".
3. Keep real API keys out of the repo. Use a git-ignored `.env`, and commit an `.env.example` with placeholder values only.
4. Make the first commit the skeleton with its tests and linter. Add one behaviour per commit after that, each with tests. Test agents with a stubbed model so the tests need no network or key.
5. Propose the commit plan below, with any changes, to the person. Wait for their sign-off, then propose `docs(work): plan TKT-06 (TKT-06)` for them to commit.
6. Work the plan one row at a time.

### The pipeline

- **Agent 1, CV Profiler:** extracts current skills, inferred level and domain baseline as validated JSON.
- **Agents 2A and 2B, run in parallel:** the Global Market Scout finds 20 global job listings, and the Local Market Scout finds 10 listings for the user's location.
- **Agent 3, Gap and Timeline Strategist:** ignores "years of experience" in listings, clusters the most frequent keywords, and scores the gap. Missing high-frequency hard skill or tool is 2 points, missing foundational paradigm is 5, and a level jump is 4 (Junior to Mid) up to 10 (Junior to Senior). The score is divided by weekly hours, and the result picks a 6-month, 1-year or 2-year roadmap. The scoring itself is plain code, not an LLM call.
- **Agent 4, Curriculum and CV Placement Architect:** sequences courses, certifications, projects, open source and events using live web search, so links are real. For each phase it also gives CV update advice: which section, which keywords to add, and how to frame it.

Its output is a JSON object with `summary` (current and target profile, duration, rationale, jobs analysed, top keywords with frequency and category) and `phases` (number, timeframe, theme, milestones with type, title, provider, cost, url, deliverable, skills gained, and a `cv_injection_guide`).

### Inputs

- **Tech track:** AI and Data, Software Engineering, Security, Cloud and Infra, Emerging and Specialised, Product and Leadership, each with the role list from the plan.
- **Seniority:** Entry/Graduate, Junior, Mid-Level, Senior, Staff/Principal, Lead/Architect.
- **Company tier:** early-stage startup, high-growth scale-up, big tech, traditional enterprise, remote-first agency.
- **Five questions:** location (country and city), work mode, weekly time commitment, upskilling budget, and learning style.

## Decisions

Signed off by the person in the first session.

- **LLM provider:** provider-agnostic through Pydantic AI's `provider:model` strings. The user picks from a short fixed list (Anthropic, OpenAI, Google Gemini), enters a model name and pastes their own key. *Why:* the person wants users to bring any AI key, and Gemini's free tier suits testing.
- **Key handling:** the key travels in a request header for one call, is never written to disk or logs, and the frontend holds it in component state only, with no `localStorage` or cookies. It never passes through the chat.
- **Job search:** Adzuna, free tier, with the operator's key in the git-ignored `.env`. *Why:* free tier, documented country and city filter, no scraping.
- **Web search:** Tavily, free tier, with the operator's key in `.env`. *Why:* Pydantic AI ships a Tavily tool.
- **Orchestration:** Pydantic AI with `asyncio.gather` for the parallel scouts. No LangGraph for now. *Why:* the pipeline is a fixed chain with one parallel step.
- **Folders:** `backend/` and `frontend/`.
- **Tooling:** backend `pytest` + `pytest-asyncio`, `ruff`, `uv`; frontend `vitest` + Testing Library, `eslint` + `prettier`.
- **Gap scoring:** `adjusted = raw ÷ hours factor`. Factors: under 5 h = 0.5, 5 up to 10 h = 1.0, 10 up to 20 h = 1.5, 20 h or more = 2.0, with boundary values in the higher band. Raw is 2 per missing skill or tool, 5 per missing paradigm, and 4 to 10 for the level jump. Precedence is top-down: 15 or more is 2 years, else 7 or more is 1 year, else 6 months. Computed in plain code.
- **CV validity:** under 100 whitespace-separated words after extraction is rejected before any agent runs. Zero matches against a bundled tech-term lexicon warns but doesn't block.
- **Commit plan:** the 14 rows as written. If commit 12 (upload plus key entry) grows, split it through a plan change.

## Commit plan

| # | Commit message | What it covers | Status |
| --- | --- | --- | --- |
| 1 | `chore(backend): scaffold the FastAPI app (TKT-06)` | Backend folder with FastAPI, a health endpoint, the test runner, the linter, and `.env.example` | |
| 2 | `feat(backend): add pydantic schemas (TKT-06)` | Models for preferences, CV profile, job listings, and the roadmap output JSON, with tests | |
| 3 | `feat(backend): parse CVs from PDF and DOCX (TKT-06)` | `pymupdf` and `python-docx` text extraction, and the too-short or no-skills warning, with tests | |
| 4 | `feat(backend): add the CV profiler agent (TKT-06)` | Agent 1, tested against a stubbed model | |
| 5 | `feat(backend): add the job search client (TKT-06)` | Wrapper around the signed-off job-search API with error handling, tested with recorded responses | |
| 6 | `feat(backend): add the market scout agents (TKT-06)` | Agents 2A and 2B, run in parallel, tested with stubs | |
| 7 | `feat(backend): score the skill gap and timeline (TKT-06)` | Agent 3's deterministic scoring and keyword clustering, with tests for the 6-month, 1-year and 2-year cases | |
| 8 | `feat(backend): add the curriculum architect agent (TKT-06)` | Agent 4 with the web-search tool, producing milestones and CV injection guides | |
| 9 | `feat(backend): chain the agents behind /roadmap (TKT-06)` | Pipeline orchestration and the upload endpoint, with an end-to-end test on stubs | |
| 10 | `chore(frontend): scaffold the Next.js app (TKT-06)` | Frontend folder with Next.js, Tailwind, the test runner and the linter | |
| 11 | `feat(frontend): add the preferences form (TKT-06)` | The three-part selector and the five quick questions | |
| 12 | `feat(frontend): add CV upload and API key entry (TKT-06)` | File upload, key handling that never stores or logs the key, and error states | |
| 13 | `feat(frontend): render the roadmap timeline (TKT-06)` | Phase cards, type badges, and the collapsible CV injection drawer with copyable keyword chips | |
| 14 | `docs: add run instructions (TKT-06)` | README section on setup, keys and running both halves | |

## Post-commit testing

| # | Check | How | Status |
| --- | --- | --- | --- |
| 1 | All checks pass | Run the backend and frontend tests and linters, and see them pass | |
| 2 | PDF CV gives a roadmap | Upload a real PDF CV with preferences, and see a roadmap whose JSON passes the schema | |
| 3 | DOCX CV works | Repeat with a DOCX CV | |
| 4 | Scanned or empty CV is rejected | Upload an image-only PDF, and see a clear warning before any agent runs | |
| 5 | Timeline rule | Run three profiles (same-domain, cross-domain, cross-domain with low hours) and see 6 months, 1 year and 2 years | |
| 6 | Links are real | Open five links from a roadmap, and confirm none is dead | |
| 7 | Bad or missing API key | Run with no key and with a wrong key, and see a clear error and no crash | |
| 8 | Key is never stored | Search logs, repo and browser storage for the key after a run, and find nothing | |
| 9 | Full run in the browser | Fill the form, upload a CV, and see phases, badges and the CV injection drawer | |

## Acceptance criteria

- [x] The person signed off the LLM provider and how users supply their key
- [x] The person signed off the job-search and web-search APIs
- [x] The person signed off the orchestration library, the folder names and the tooling
- [x] The person signed off the gap scoring thresholds
- [x] The person signed off the commit plan
- [ ] A user can upload a PDF or DOCX CV, set preferences, and get a phased roadmap
- [ ] A CV with too little text gets a clear warning and no agent runs
- [ ] The roadmap duration follows the signed-off scoring rules, not an LLM's opinion
- [ ] Every agent returns validated JSON, so the frontend never parses free text
- [ ] No API key or CV is stored or committed

## Stretch goals

Once every acceptance criterion is met, a team with time to spare can add these to the plan as new commits:

- Stream live agent progress to a horizontal stepper using server-sent events
- A keyword heatmap banner, green for skills already on the CV and amber for gaps
- Smooth timeline reveals with Framer Motion
- Saved roadmaps, or a downloadable PDF of the roadmap
