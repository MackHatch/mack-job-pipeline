# Google Sheet headers (row 1)

Create a sheet tab named `Applications` (or change the n8n Google Sheets node to match).

## Recommended headers (match the live workflow mappings)

Put these in row 1, one column each:

```
receivedAt | jobUrl | company | title | fitScore | applyPriority | liveCodingRisk | stackGaps | coverLetter | whyMe | recruiterMessage | interviewFormatQuestion | status | notes
```

Optional extras if you map them:

```
runId | location | seniority | stackMatch | resumeHeadline | risks | whyFit | reasoning
```

## Mapping tips (n8n → Sheets)

| Sheet column | Typical expression / value |
|--------------|----------------------------|
| `receivedAt` | `{{ $('Normalize Input').item.json.receivedAt }}` |
| `jobUrl` | `{{ $('Normalize Input').item.json.jobUrl }}` |
| `notes` | `{{ $('Normalize Input').item.json.notes }}` |
| `company`, `title`, `fitScore`, … | `{{ $json.<field> }}` from **Parse Draft** |
| `stackGaps` / arrays | `{{ JSON.stringify($json.stackGaps) }}` or `.join('\\n')` |
| `whyMe` | `{{ $json.whyMeBullets.join('\\n') }}` |
| `status` | fixed text `draft_ready` (expression off) |

Low-fit skips can use `status = skipped_low_fit` if you append from the false branch later.
