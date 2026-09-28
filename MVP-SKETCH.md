# AI Job Application Pipeline — MVP Sketch

**Goal:** One n8n workflow that turns a job URL into a scored fit + draft application package, with human review before anything is sent.  
**Why it belongs on your GitHub:** Demonstrates n8n orchestration, structured LLM outputs, eval/safety gates, and a real workflow you use — AI SDLC in practice, not slides.

**Out of scope for MVP:** Auto-submit to employers, LinkedIn scraping, PDF résumé generation, daily board crawling.

---

## Architecture (one sentence)

You paste a job URL → n8n fetches the posting → an LLM scores fit against your résumé profile (JSON schema) → if above threshold, a second LLM drafts cover letter + ATS bullets → results land in a Sheet and you get a digest → *you* decide what to send.

```
[Manual Trigger / Form]
        │
        ▼
[Normalize input] ── jobUrl, notes?, source?
        │
        ▼
[HTTP Request] ── fetch posting HTML/text
        │
        ▼
[Extract text] ── strip nav/boilerplate (simple for MVP)
        │
        ▼
[LLM: Score fit] ── structured JSON (often nested under output[].content[].text)
        │
        ▼
[Code: Parse Fit] ── flatten fitScore, gaps, liveCodingRisk, …
        │
        ├─ score < 7 ──► Skip draft (optional: still log to Sheet)
        │
        ▼ score ≥ 7
[LLM: Draft package] ── letter + why-me + recruiter blurb
        │
        ▼
[Code: Parse Draft] ── flatten draft JSON; merge with fit fields
        │
        ▼
[Google Sheets append] ── status draft_ready
        │
        ▼
[Gmail digest] ── YOU review; nothing auto-applies
```

---

## n8n node list (build order)

| # | Node | Purpose |
|---|------|---------|
| 1 | **Form Trigger** or **Webhook** | Input: `jobUrl` (required), `notes` (optional), `companyHint` (optional) |
| 2 | **Set** | Normalize fields; add `runId`, `receivedAt` |
| 3 | **HTTP Request** | GET `jobUrl`; follow redirects; user-agent header |
| 4 | **Code** (or HTML Extract) | Pull main text; truncate to ~12–15k chars for the model |
| 5 | **OpenAI / HTTP → LLM** | Fit scorer with JSON schema / response_format |
| 6 | **IF** | `fitScore >= 7` (tune later) |
| 7 | **OpenAI / HTTP → LLM** | Draft package (only on pass) |
| 8 | **Google Sheets** | Append row (always, including low-score skips) |
| 9 | **Gmail / Slack** | Digest to you with Sheet link + score + draft |

Optional later: Error Trigger → Slack “pipeline failed for URL X”.

---

## Data contracts (treat like Zod schemas)

### Input
```json
{
  "jobUrl": "https://...",
  "notes": "Optional: recruiter name, referral, must-mention Vue",
  "companyHint": "Optional free text"
}
```

### Fit score output (LLM 1) — require every field
```json
{
  "fitScore": 8,
  "title": "Senior Software Engineer",
  "company": "Example Co",
  "location": "Remote Canada",
  "seniority": "senior",
  "stackMatch": ["C#", ".NET", "React", "SQL Server"],
  "stackGaps": ["Kubernetes", "GCP"],
  "liveCodingRisk": "unknown | likely | unlikely",
  "whyFit": ["bullet", "bullet"],
  "risks": ["bullet"],
  "applyPriority": "high | medium | low | skip",
  "reasoning": "2-4 sentences grounded only in posting + profile"
}
```

### Draft package output (LLM 2)
```json
{
  "resumeHeadline": "one line",
  "coverLetter": "350-500 words, first person as Mackenzie Hatcher",
  "whyMeBullets": ["...", "..."],
  "recruiterMessage": "3 lines max",
  "interviewFormatQuestion": "one sentence to ask recruiter about live coding vs take-home"
}
```

### Sheet columns
`runId | date | url | company | title | fitScore | applyPriority | liveCodingRisk | stackGaps | coverLetter | whyMe | recruiterMessage | status`

`status` defaults to `draft_ready` or `skipped_low_fit`. You change to `applied` / `passed` manually.

---

## Prompt rules (AI SDLC — put these in the system prompts)

1. **Grounding:** Use only the posting text + the fixed résumé profile block. Never invent employers, metrics, or skills.
2. **Honesty:** Surface stack gaps and years gaps; do not inflate 6+ to 8+.
3. **Canada bias:** Prefer remote Canada / Manitoba-eligible; flag US-only remote.
4. **Live-coding flag:** If the posting mentions HackerRank, CoderPad, pair programming, live coding — set `liveCodingRisk: likely`.
5. **Human gate:** Drafts are suggestions. The workflow must never email an employer.

Embed a short **résumé profile** as a pinned Set-node string (or n8n credential/static data): stack, years, MITT/InTouchCX bullets, AI agent project, Winnipeg, contact — same facts as your résumé.

---

## Eval cases (show interviewers you test agents)

Keep 3–5 saved fixtures (URL or pasted JD text → expected score band):

| Case | Expect |
|------|--------|
| Strong .NET + React remote Canada | score ≥ 8, applyPriority high |
| Pure Java / Spring only | score ≤ 4, skip |
| 8+ years Angular + AWS heavy | medium score, risks called out |
| Mentions live pair programming | liveCodingRisk likely |
| Hallucination trap: JD with fake metric | draft must not invent your metrics |

Run fixtures after any prompt change. Document results in `evals/README.md`.

---

## Repo layout (portfolio)

```
ai-job-application-pipeline/
  README.md                 # problem, architecture diagram, demo GIF later
  docs/MVP-SKETCH.md        # this file
  n8n/workflow.json         # exported workflow
  prompts/fit-scorer.md
  prompts/draft-package.md
  profile/resume-profile.md # sanitized profile fed to LLMs
  evals/cases/*.json
  evals/README.md
```

README talking points for interviews:
- Why n8n (orchestration, retries, human-in-the-loop)
- Structured outputs / schema validation
- Eval harness before prompt edits
- Explicit non-goals (no auto-apply)

---

## Build order (tonight → next sessions)

**Tonight (sketch → stub):** Finalize this doc; write `resume-profile.md`; draft both prompts; define Sheet headers.  
**Next build session:** Stand up n8n (local Docker or n8n cloud); Form → HTTP → Code → fake LLM (Set node with sample JSON) → Sheet. Prove the pipe without burning tokens.  
**Then:** Wire real LLM with JSON mode; IF threshold; second draft call; email digest.  
**Polish:** Eval cases; README; 60-second Loom for applications.

---

## Interview story (90 seconds)

“I built an n8n pipeline that takes a job URL, scores fit against my résumé with a structured schema, and drafts a cover letter only when the score clears a threshold. Everything lands in a sheet for my review — nothing auto-applies. I treat prompts like code: fixtures for strong-fit, bad-fit, and live-coding flags, and I refuse inventing skills. It’s the same discipline I’d use putting AI into a product SDLC.”

---

## Decisions to lock before coding

1. **n8n host:** local Docker vs n8n Cloud  
2. **LLM:** OpenAI API (you already know it) vs other  
3. **Store:** Google Sheets (simplest) vs Notion  
4. **Notify:** Email vs Slack  
5. **Threshold:** start at `7`, tune after 10 real jobs

---

## Locked decisions (v1 — interview / GitHub optimized)

| Choice | Decision | Why |
|--------|----------|-----|
| n8n host | **Local Docker** | Self-host story; export workflow.json; $0 host |
| LLM | **OpenAI** (cheap model for fit score; stronger for drafts) | Matches existing agents/Zod work; coherent interview narrative |
| Store | **Google Sheets** | Fast demo UX; eng depth lives in repo/evals, not the DB |
| Notify | **Email** to mack.hatcher1@outlook.com | Human-in-the-loop proof |
| Non-negotiables for repo | workflow export, prompts, profile, eval fixtures, README, no auto-apply | Senior AI SDLC signal |

Deferred: n8n Cloud, Notion, Claude, Postgres/SQLite (possible v2).
