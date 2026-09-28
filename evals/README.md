# Evals

After changing prompts, run each case through the fit-scorer (manually in n8n or via API) and check `expect` bands.

| Case | Expect |
|------|--------|
| strong-dotnet-react | fitScore ≥ 8, applyPriority high/medium |
| bad-fit-java | fitScore ≤ 4, skip/low |
| live-coding-flag | liveCodingRisk = likely |
