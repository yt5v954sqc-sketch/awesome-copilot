---
name: suggest-awesome-github-copilot-agents
description: 'Suggest relevant GitHub Copilot Custom Agents files from the awesome-copilot repository based on current repository context and chat history, avoiding duplicates with existing custom agents in this repository, and identifying outdated agents that need updates.'
---

description: Analyze the current repository and suggest relevant GitHub Copilot Custom Agents from the awesome-copilot repository. Compares local agents against remote versions, identifies gaps, detects outdated files, and presents a structured recommendation table.
tools:

fetch

githubRepo

runInTerminal

todos

readFile

listDirectory
applyTo: "**/.github/agents/*.agent.md"

Suggest Awesome GitHub Copilot Custom Agents
Analyze current repository context and suggest relevant Custom Agents from
github/awesome-copilot
that are absent or outdated in this repository.

Tool Routing
Use the correct tool per step — do not substitute:

Step	Tool	Reason
Fetch README.agents.md	fetch	Raw HTTP — no GitHub auth needed
Fetch remote agent file	fetch (raw URL)	Retrieve exact file bytes for diff
List local agent files	listDirectory + readFile	Read local .github/agents/
Download/update agent	runInTerminal with curl -fsSL	Ensures full content, handles redirects
Track install progress	todos	Ordered checklist for multi-file installs
Browse repo file tree	githubRepo	Only when directory listing is needed
Execution Pipeline
Run steps in the order below. Steps 2–5 may be parallelised.

STEP 1 — Fetch Upstream Catalogue
Fetch the upstream agent list using the fetch tool:

text
URL: https://raw.githubusercontent.com/github/awesome-copilot/main/docs/README.agents.md
Parse every Markdown table row into a structured record:

text
{
  title:       string          // display name from table column 1
  filename:    string          // kebab-case + ".agent.md" (derive if not explicit)
  description: string          // table column 2
  mcp_servers: string[]        // table column 3, split on comma; [] if empty
  remote_url:  string          // https://github.com/github/awesome-copilot/blob/main/agents/<filename>
  raw_url:     string          // https://raw.githubusercontent.com/github/awesome-copilot/main/agents/<filename>
}
Error handling: If the fetch returns non-200, retry once. On second failure,
output: ⚠️ Could not reach awesome-copilot — results may be incomplete. and
continue with an empty upstream catalogue so local-only analysis still runs.

STEP 2 — Inventory Local Agents
List all files matching .github/agents/*.agent.md.

For each file, parse the YAML front matter block (between --- delimiters).

Extract the following fields (record null if absent):

text
# Fields that matter for comparison
description: <string>
tools:        <string[]>
mode:         <string>
applyTo:      <string>
Build a map keyed by normalised filename (lowercase, hyphens, .agent.md suffix).

If .github/agents/ does not exist, record zero local agents and skip Steps 3–5.

STEP 3 — Fetch Remote Counterparts
For every filename in the local inventory, construct its raw_url and fetch it
with the fetch tool. Collect the raw content for each.

404 → mark as local-only (not from awesome-copilot; skip version comparison).
Network error → mark as UNKNOWN (cannot determine status; flag in output table).

STEP 4 — Compute Version Status
Compare each local file against its remote counterpart using this decision tree:

text
local file exists?
├─ NO  → status = ❌ NOT INSTALLED
└─ YES → fetch remote
         ├─ remote NOT FOUND (404)  → status = ✅ LOCAL ONLY (custom, not from upstream)
         ├─ fetch ERROR             → status = ❓ UNKNOWN
         └─ remote FOUND
              ├─ byte-for-byte identical → status = ✅ UP-TO-DATE
              └─ content differs
                   ├─ diff front matter fields (description, tools, mode, applyTo)
                   ├─ diff body content (instructions, examples)
                   └─ status = ⚠️ OUTDATED
                      record specific_diffs[] listing each changed field/section
specific_diffs format:

text
- tools: added ['web/fetch'], removed ['fetch']
- description: wording changed (first 80 chars of each shown)
- body: ~12% content change (section "Usage Examples" modified)
STEP 5 — Score Relevance for Uninstalled Agents
For each upstream agent not present locally, compute a relevance score (0–10)
based on repository signals detected in STEP 6.

Signal	Score bonus
Language match (.cs → .NET agents, .py → Python agents, etc.)	+3
Framework file present (angular.json, next.config.*, etc.)	+3
Cloud provider config detected (azure*, terraform*, etc.)	+2
Agent fills an explicit gap in local library (no test agents, no docs agents)	+2
Agent requires MCP server not yet configured in repo	−1
Agent is a novelty/persona with no clear repo alignment	−2
Only include agents with score ≥ 3 in the output table. Suppress lower-scored agents.

STEP 6 — Analyse Repository Context
Collect the following signals before scoring (used in STEP 5):

File-system signals:

Languages: scan extensions (.cs, .py, .js, .ts, .go, .rs, .java, etc.)

Frameworks: package.json → dependencies, *.csproj, go.mod, Cargo.toml, pom.xml

Cloud: azure*, terraform*, bicep*, *.tf, *.bicep, k8s/, helm/

CI/CD: .github/workflows/, Jenkinsfile, .gitlab-ci.yml

Testing: *test*, *spec*, jest.config.*, pytest.ini, xunit*

Docs: docs/, ADR/, *.md count > 10

Chat history signals:

Extract technology names, pain points, or workflow keywords from recent turns.

Boost score for any upstream agent that directly addresses a mentioned topic.

STEP 7 — Build Output Table
Output exactly one Markdown table. No prose before or after the table except
the two-line summary block defined below.

Summary block (above table):
text
📦 Local agents found: <N>  |  ✅ Up-to-date: <N>  |  ⚠️ Outdated: <N>  |  ❌ Not installed (shown): <N>
🔍 Repository signals: <comma-separated list of detected languages/frameworks>
Table schema:
Column	Content
Awesome-Copilot Agent	Linked filename → https://github.com/github/awesome-copilot/blob/main/agents/<filename>
Description	description field from remote front matter (truncated to 120 chars)
Status	✅ Up-to-date · ⚠️ Outdated · ❌ Not installed · ❓ Unknown · 🔒 Local Only
Local File	Filename if installed, else —
Relevance / Diff Notes	For ❌: one-sentence rationale (score ≥ 3 required). For ⚠️: list specific_diffs. For ✅: —
Row ordering:
⚠️ Outdated (update recommended — highest priority)

❌ Not installed, sorted by relevance score descending

✅ Up-to-date (collapsed at bottom, no rationale needed)

🔒 Local Only (custom, not from upstream)

STEP 8 — Await User Instruction
STOP HERE. Do not download, modify, or create any files.

End with exactly this prompt:

Reply with the agent filenames you want to install or update, and I will
download them to .github/agents/ without modifying content.

STEP 9 — Install / Update (Only When Directed)
Trigger only when the user explicitly names one or more agents to install or update.

For each requested agent:

Add a todos entry: [ ] Install <filename>

Construct the raw URL:
https://raw.githubusercontent.com/github/awesome-copilot/main/agents/<filename>

Run in terminal:

bash
mkdir -p .github/agents
curl -fsSL "<raw_url>" -o ".github/agents/<filename>"
Verify file exists and is non-empty (wc -c).

Mark todos entry complete: [x] Install <filename>

Report result inline: ✅ <filename> saved to .github/agents/

NEVER modify file content. Write bytes exactly as received.
NEVER install agents not explicitly requested.
If curl returns non-zero exit code, report the error and leave the file untouched.

Guard Rails
DO NOT install or overwrite any file before STEP 9 is explicitly triggered.

DO NOT hallucinate filenames — only reference agents confirmed in the upstream catalogue.

DO NOT include agents with relevance score < 3 in the output table.

DO NOT merge or edit file content during install — raw bytes only.

DO NOT run terminal commands in STEP 9 unless curl is available (which curl).
If unavailable, fall back to fetch tool and write via writeFile.

DO NOT output anything beyond the summary block and the table until STEP 8 is complete.

Status Icon Reference
Icon	Meaning
✅	Installed and byte-identical to upstream
⚠️	Installed but content differs from upstream — update available
❌	Not installed — recommended based on relevance score
❓	Status unknown — fetch failed; manual check needed
🔒	Local custom agent — not present in awesome-copilot upstream
