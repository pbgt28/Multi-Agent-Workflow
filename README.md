# Multi-Agent Company Research Pipeline

An n8n workflow that takes a company URL, researches the company, and writes a profile report. A separate reviewer agent then scores the report out of 10. If the reviewer isn't satisfied, the report goes back for another revision. Approved results are logged to a Google Sheet.

I built it to see how far an orchestrator agent could coordinate other agents on its own, and to add an automated check on the output instead of trusting the first draft.

## How it works

1. A POST request hits the webhook with the task: a company URL and an instruction to write a concise company profile.
2. The Initialize Task node sets up the task ID, the task text, and the revision counter.
3. The Orchestrator Agent reads the task and calls three sub-agents as tools: researcher, analyst, and writer.
4. The Reviewer Agent reads the finished report and returns a score and a verdict.
5. A Code node (Parse Review) turns the reviewer's reply into structured fields.
6. An IF node, "Approved or Max Revisions?", decides what happens next.
   - If the report is approved, or the maximum number of revisions has been reached, the Finalize Result node prepares the output and the Log to Google Sheets node appends a row.
   - Otherwise, the Prepare Revision node adds the reviewer's feedback to the task and sends it back to the orchestrator for another pass.

## Agents

The workflow has five agents. Each one has its own chat model node.

| Agent | Role |
|---|---|
| Orchestrator | Breaks down the task, calls the other agents, and keeps conversation memory across the run |
| Researcher | Gathers background on the company, using a Wikipedia tool |
| Analyst | Pulls the research into findings about what the company does and sells |
| Writer | Writes the final profile report |
| Reviewer | Scores the report and decides whether it passes |

All five use OpenAI chat models. [add the exact model name(s) here]

## Setup

You need an n8n instance (I used n8n Cloud), an OpenAI API key, and a Google account.

1. Import the workflow JSON from this repo into n8n.
2. Add an OpenAI credential and select it on each of the five model nodes.
3. Create a Google Sheet with this header row: task_id, task, final_output, review_score, status, timestamp.
4. Add a Google Sheets credential in n8n, then open the Log to Google Sheets node and select your document and sheet. The operation is Append Row.
5. Set the maximum number of revisions. [add the number you configured and where it lives]
6. Save the workflow.

## Running it

Use the webhook's test URL while building, and the production URL once the workflow is active. Send a POST request like this:

```bash
curl -X POST "[your webhook URL]" \
  -H "Content-Type: application/json" \
  -d '{"[field name]": "Write a concise company profile for https://www.example.com"}'
```

Each finished run adds one row to the sheet.

## Output log

| Column | What it holds |
|---|---|
| task_id | A timestamp-based ID for the run |
| task | The input given to the workflow |
| final_output | The report that was approved |
| review_score | The reviewer's score out of 10 |
| status | approved |
| timestamp | When the row was logged |

## Results so far

I ran the workflow on 5 companies and timed every run from the n8n Executions tab.

| Company | Review score (out of 10) | Run time |
|---|---|---|
| Infosys | 8 | 48.1 s |
| SAP | 8 | 48.3 s |
| American Express | 9 | 56.8 s |
| Microsoft | 9 | 46.7 s |
| Wipro | 9 | 55.0 s |

All 5 runs ended with status approved. The average review score was 8.6 out of 10, and the average run time was about 51 seconds.

## Limitations

- The reviewer is an LLM, so its score measures how well the report meets the reviewer's criteria, not whether the facts are correct.
- The sheet doesn't record how many revisions each run needed, so I can't say how many reports passed on the first try.
- Five runs is a small sample, and all of them were approved, so these numbers say little about how often the reviewer sends a report back.
- The researcher uses Wikipedia only, so coverage of smaller or newer companies will be thin.

## Possible next steps

- Add a revisions column to the log.
- Give the researcher a web search or page-fetch tool.
- Spot-check a sample of reports by hand and compare the results with the reviewer's scores.
