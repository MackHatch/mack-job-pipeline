# AI Job Application Pipeline

n8n workflow that scores a job posting against Mackenzie Hatcher’s résumé and drafts an application package when the fit clears a threshold. **Nothing auto-applies** — drafts land in Google Sheets and an email digest for human review.

## Stack (v1)

| Piece | Choice |
|-------|--------|
| Orchestrator | n8n (local Docker) |
| LLM | OpenAI (cheap model for scoring; stronger for drafts) |
| Store | Google Sheets |
| Notify | Email |
| Timezone | America/Winnipeg |

## Quick start

```bash
cd ai-job-application-pipeline   # or this folder
docker compose up -d
```

Open http://localhost:5678 — create owner account on first launch.

Import `n8n/workflow.json` (File → Import). Add credentials:

1. **OpenAI** API key  
2. **Google Sheets** OAuth (sheet with headers from `docs/sheet-headers.md`)  
3. **Email** (SMTP or Gmail/Outlook node — use `mack.hatcher1@outlook.com`)

Paste system prompts from `prompts/*.system.txt`. Load candidate facts from `profile/resume-profile.md` into a Set node (or read the mounted file).

## Flow

1. Form/Webhook: `jobUrl`, optional `notes`  
2. Fetch posting HTML → extract text  
3. LLM fit score (JSON schema)  
4. If `fitScore >= 7` → LLM draft package  
5. Append Google Sheet row  
6. Email digest to you  

## AI SDLC practices

- Structured JSON outputs (see prompts)  
- Eval fixtures in `evals/cases/` — run after prompt changes  
- Explicit gaps (years, cloud, Angular, live-coding risk)  
- Human-in-the-loop before any employer contact  

## Interview story (90s)

See `MVP-SKETCH.md`.

## License

Personal project — not affiliated with any employer.
