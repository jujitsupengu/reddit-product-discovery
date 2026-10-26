# Reddit Product Idea Discovery

An n8n workflow that searches Reddit for posts where people in specific UK B2B verticals describe a problem they would pay to have solved. Claude scores each post and writes the strong ones to a Google Sheet.

## How it works

1. **Trigger.** A schedule (Monday and Thursday at 06:00, instance timezone) or the manual trigger.
2. **Configuration.** One Set node holds every tuning value (see below).
3. **Load Vertical Taxonomy.** A Code node embeds 19 verticals (law firms, accounting, lettings, insurance brokers, PE/VC fund ops and others). Each has subreddits, pain phrases, buyer personas, trigger events, existing tools, top opportunities and a context card for the prompt. Each vertical has an `enabled` flag.
4. **Build Search Matrix.** Crosses each enabled vertical's subreddits with its top pain phrases to make Reddit search URLs, then splits them into chunks.
5. **Chunk Loop.** For each chunk: start an Apify `trudax~reddit-scraper-lite` run, poll every 15s until it finishes, and fetch the results.
6. **Filter.** Trim post bodies, apply engagement floors, drop URLs already in the sheet, tag each post with its vertical, and apply the domain, noise-phrase and buyer-signal filters.
7. **Rank and cap.** Rank candidates (engagement-blended) and stop at the Claude call budget for the run.
8. **Claude Analysis.** Each post goes to the Anthropic Messages API with its vertical's context card, in a step-by-step reasoning prompt that returns structured JSON.
9. **Gate and write.** Posts that pass the score or opportunity gate are appended or updated in the sheet, matched on `URL`.
10. **Progress and debug rows.** Each chunk writes a progress row, and each run writes a debug summary row.

## Run modes

`runMode` in Configuration picks the search profile:

| Value | Behaviour |
|---|---|
| `auto` (or empty) | Manual executions run as `backtest`, scheduled ones as `weekly` |
| `backtest` | `backtestSort` / `backtestTimeWindow` / `backtestMaxPosts` / `backtestMaxClaudeCalls` (default: top posts of the past year) |
| `weekly` | `weeklySort` / `weeklyTimeWindow` / `weeklyMaxPosts` / `weeklyMaxClaudeCalls` (default: relevance, past week) |

The file ships with `runMode: backtest`, `debugMode: true` and `runLabel: p4s-smoke`. Set `runMode` to `auto` and `debugMode` to `false` for scheduled use.

## Setup

1. **Create credentials in n8n.** All credential IDs in `workflow.json` are `__CREDENTIAL_PLACEHOLDER__`. After import, open each node listed here and select your own credential:

   | Credential type | Used by | What it holds |
   |---|---|---|
   | HTTP Query Auth | Start Apify Scraper, Check Run Status, Get Scraper Results | Query parameter `token` = your Apify API token |
   | HTTP Header Auth | Claude Analysis (FP/AGoT) | Header `x-api-key` = your Anthropic API key |
   | Google Sheets OAuth2 | Load Already-Processed URLs, Append or update row in sheet, Append Debug Summary, Write Progress Row | Your Google account |

2. **Create the Google Sheet** with three tabs:
   - `v3`: the results. Headers: `Post Title`, `Vertical`, `Tier`, `Matched Opportunity`, `Opp Confidence`, `Review Tier`, `Matched Phrases`, `Path Tag`, `Problem Summary`, `Query Category`, `Product Idea`, `Outreach Hook`, `Trigger Event`, `Scores`, `Reasoning Chain`, `Boosts`, `Overall Score`, `Author`, `Subreddit`, `Upvotes`, `Comments`, `URL`, `Processed At`, `Run Mode`
   - `debug_runs`: one row per run. Headers: `Run Label`, `Execution ID`, `Started At`, `Scraped`, `Engagement Pass`, `Domain Pass`, `Claude Analyzed`, `Score Pass`, `Appended`, `Top Verticals`, `Drop Summary`, `Notes`
   - `Status`: one row per chunk. Headers: `runLabel`, `progress`, `scraped`, `leads`, `runningTotalLeads`, `elapsedSec`, `mode`, `ts`

3. **Set the sheet ID.** In the Configuration node, replace `__SET_SHEET_ID__` with the ID from your sheet's URL.

4. **Import** `workflow.json` into n8n (Workflows → Import from file), attach the credentials, and run the manual trigger once before activating the schedule.

## Configuration knobs

All of these live in the Configuration node.

| Knob | Shipped value | What it controls |
|---|---|---|
| `minUpvotes` / `minComments` | 5 / 1 | Engagement floor |
| `maxPostsPerSearch` | 2 | Posts Apify returns per search URL |
| `topKPhrasesPerVertical` | 4 | Pain phrases per vertical that become searches |
| `topKManualWorkflow` | 0 | Extra "manual workflow" phrase searches |
| `scoreThreshold` | 5.5 | Minimum overall score written to the sheet |
| `claudeModel` | `claude-sonnet-4-6` | Model for the analysis call |
| `minBodyLength` / `maxBodyLength` | 50 / 1500 | Body length kept and sent to Claude |
| `maxClaudeCalls` | 20 | Claude call cap when the mode does not set one |
| `enabledVerticalIds` | empty | Comma list to restrict verticals; empty means every enabled one |
| `domainFilterMode` | `relaxed` | Domain filter strictness. `strict` let almost nothing through in testing |
| `noisePhrases` / `buyerSignalPhrases` | see file | Comma lists that reject or promote posts |
| `broadSubreddits` | see file | Broad subreddits that need extra phrase hits to count |
| `backtestChunkSize` | 40 | Search URLs per Apify run |

## Editing verticals

`verticals-config.json` is the editable source. The `Load Vertical Taxonomy` Code node holds a copy, so after editing the JSON, paste the updated `verticals` array into that node's `CONFIG.verticals`.

## Cost

Apify charges per run and per result, and Anthropic per call. The Claude call caps (`backtestMaxClaudeCalls`, `weeklyMaxClaudeCalls`, `maxClaudeCalls`) bound the Anthropic spend per run. If Apify runs fail within seconds and return nothing, check your Apify credit balance first.

## License

MIT. See [LICENSE](LICENSE).
