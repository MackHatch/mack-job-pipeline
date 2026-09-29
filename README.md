# AI Job Application Pipeline

Local **n8n** workflow that scores a job posting against a fixed résumé profile and drafts an application package when fit clears a threshold. **Nothing auto-applies** — drafts land in Google Sheets and a Gmail digest for human review.

Built as a portfolio / interview story for AI SDLC: structured LLM outputs, eval fixtures, explicit risk fields (including live-coding), and a hard human-in-the-loop gate.

## Stack (v1)

| Piece | Choice |
|-------|--------|
| Orchestrator | n8n (local Docker) |
| LLM | OpenAI — `gpt-4o-mini` (or `gpt-4.1-mini`) for fit score; `gpt-4.1` or `gpt-4o` for drafts. Avoid o-series. |
| Store | Google Sheets |
| Notify | Gmail |
| Timezone | America/Winnipeg |

## Repo layout

```
docker-compose.yml          # n8n on http://localhost:5678
n8n/workflow.json           # scoring workflow — Job URL Form (re-import after clone)
n8n/digest-workflow.json    # weekday draft_ready digest (separate; does not apply)
profile/resume-profile.md   # locked candidate facts for prompts
prompts/
  fit-scorer.system.txt
  draft-package.system.txt
docs/sheet-headers.md       # row-1 headers for the Sheet
evals/cases/                # fixtures after prompt changes
MVP-SKETCH.md               # architecture + 90s interview pitch
```

## Quick start

```bash
cd mack-job-pipeline   # or your clone path
docker compose up -d
```

Open http://localhost:5678 — create the owner account on first launch.

1. **Import** `n8n/workflow.json` (⋯ → Import from File).
2. Recreate credentials (exports do not include secrets):
   - **OpenAI** API key
   - **Google Sheets OAuth2** (Sheets + Drive APIs; redirect `http://localhost:5678/rest/oauth2-credential/callback`)
   - **Gmail OAuth2** (Gmail API + `gmail.send` scope; same redirect)
3. Point the Sheets node at your spreadsheet / `Applications` tab (headers in `docs/sheet-headers.md`).
4. Confirm OpenAI nodes use the system prompts under `prompts/` and the profile under `profile/resume-profile.md`.
5. Open the Form trigger URL, paste a job URL, optional notes → Execute.

## Flow (as built)

1. **Job URL Form** — `jobUrl`, optional `notes`
2. **Normalize Input** — adds `receivedAt` (and keeps `jobUrl` / `notes`)
3. **Fetch Job Posting** — HTTP GET
4. **Extract Text** — HTML → plain text (+ prior fields)
5. **Fit Score (OpenAI)** — structured fit JSON (often returned inside `output[0].content[0].text`, sometimes fenced)
6. **Parse Fit (Code)** — flatten to top-level `fitScore`, `stackMatch`, etc.
7. **IF** — `fitScore >= 7`
8. **Draft Package (OpenAI)** — cover letter + ATS bullets + recruiter blurb + interview-format question
9. **Parse Draft (Code)** — flatten draft JSON; merge with fit fields
10. **Google Sheets** — append row (`status = draft_ready`)
11. **Gmail** — digest to you for review

Low-fit branch: skip draft (optionally still log to the sheet later).

Open `draft_ready` rows are listed again on a weekday schedule by a **separate** workflow. See [Draft digest (scheduled)](#draft-digest-scheduled).

## Draft digest (scheduled)

Separate workflow from the Job URL Form pipeline. It only **reads** the sheet and emails you. It does not change `status`, it does not call the LLM, and it does not apply to any job.

1. **Import** `n8n/digest-workflow.json` (⋯ → Import from File). Leave `n8n/workflow.json` as the human URL entry path.
2. Re-select the same **Google Sheets** and **Gmail** credentials if they do not bind on import (exports do not include secrets). The Sheets node uses the same spreadsheet and `applications` tab (`gid=0`) as the scoring workflow. Headers: `docs/sheet-headers.md`.
3. Gmail **To** is `mack.hatcher1@outlook.com` on **Send Digest**. Change that field if you want a different inbox.
4. Workflow timezone is **America/Winnipeg** (set in the export). `docker-compose.yml` also sets `GENERIC_TIMEZONE` and `TZ` to that zone.
5. **Publish** the workflow. It is inactive in the export so it does not fire before credentials are checked. The Schedule Trigger cron is `0 8,18 * * 1-5`: weekdays at **08:00** and **18:00**.

**Keep draft_ready** keeps rows whose `status` is exactly `draft_ready` (case-sensitive, the same value the scoring workflow writes). **Build Digest** turns those rows into one plain-text email: company, title, fitScore, applyPriority, liveCodingRisk, jobUrl, receivedAt, and about the first 200 characters of coverLetter (whitespace collapsed). Higher `fitScore` is listed first.

**No `draft_ready` rows → no email.** The filter emits nothing, so Gmail does not run. An empty Applications tab does the same.

The schedule fires only while local Docker n8n is running. If the laptop is asleep or the container is stopped, that slot is missed — nothing is queued to send later. The same open drafts are listed again at the next run until you change `status` in the sheet (for example to `applied`).

## Google Cloud OAuth (local Docker)

Your `docker-compose.yml` already sets `WEBHOOK_URL=http://localhost:5678/`.

1. Enable **Google Sheets API**, **Google Drive API**, and **Gmail API**.
2. OAuth client type **Web application**.
3. Authorized redirect URI (exact):

   `http://localhost:5678/rest/oauth2-credential/callback`

4. Under Data Access / scopes, include at least:

   - `https://www.googleapis.com/auth/spreadsheets`
   - `https://www.googleapis.com/auth/drive.file`
   - `https://www.googleapis.com/auth/gmail.send`

5. Consent screen in **Testing** → add your Google account as a test user.

## Implementation notes (worth keeping)

- OpenAI “Message a model” output is nested. Parse with **Code → Run Once for All Items**, read `output[0].content[0].text` (handle `output` as array), strip/find `{...}`, `JSON.parse`. Turn **off** Continue On Fail so parse errors surface.
- After Parse Fit, IF uses `{{ $json.fitScore }}` **≥ 7** (number).
- `jobUrl` / `receivedAt` / `notes` live on **Normalize Input** (and Extract Text). Sheets mappings for those fields should reference `$('Normalize Input')` if Parse Draft does not re-merge them.
- Never commit API keys or OAuth client secrets. Re-export `workflow.json` after graph changes; scrub if anything secret leaked into the file.

## AI SDLC practices

- Structured JSON contracts in `prompts/`
- Eval fixtures in `evals/cases/` — re-run after prompt edits
- Explicit gaps: years, cloud, Angular, **liveCodingRisk**
- Human review only — no auto-apply to employers

## Interview story (90s)

See `MVP-SKETCH.md`.

## License

Personal project — not affiliated with any employer.
