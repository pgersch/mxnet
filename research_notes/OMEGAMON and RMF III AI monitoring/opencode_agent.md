# OpenCode as an Unattended, Interval-Driven Monitoring Agent (OMEGAMON / RMF III context)

Research date: 2026-10-17. Primary sources: opencode.ai/docs (fetched live), github.com/anomalyco/opencode, github.com/zowe/zowe-mcp, docs.openhands.dev. Where a fact comes from a third-party aggregator rather than a primary source, that is flagged.

## 1. How to run OpenCode non-interactively (run, serve + SDK, JSON output, exit codes, permissions)

### Takeaway
OpenCode has two headless modes. `opencode run "prompt"` is a one-shot CLI that can emit raw JSON events. `opencode serve` is an HTTP server (default 127.0.0.1:4096) driven by REST/OpenAPI or the `@opencode-ai/sdk` TypeScript SDK, which supports JSON-Schema structured output. To run unattended, set `permission` to explicit `allow`/`deny` values, or pass `--auto`, which approves everything not explicitly denied. Exit codes are not documented, and one open issue reports intermittent hangs, so wrap each run in an external timeout.

### Cited Findings
**Project identity / repo location**
- The project was historically at github.com/sst/opencode (by Anomaly Innovations, the makers of SST). The active issue tracker and the GitHub Action now live under `anomalyco/opencode`. No announcement of the move was found. The docs reference `uses: anomalyco/opencode/github@latest`. — [horadecodar (sst/opencode, Anomaly Innovations)](https://horadecodar.com.br/sst-opencode/); [anomalyco/opencode issues](https://github.com/anomalyco/opencode/issues); [OpenCode GitHub docs](https://opencode.ai/docs/github/)

**`opencode run` (one-shot CLI)**
- `run` is meant for "scripting, automation, or when you want a quick answer without launching the full TUI." — [OpenCode CLI docs](https://opencode.ai/docs/cli/)
- Flags for `opencode run [message..]` (from the CLI docs):
  - `--model/-m provider/model`
  - `--agent <name>`
  - `--format default|json`. JSON emits "raw JSON events" for parsing.
  - `--file/-f` attaches files.
  - `--continue/-c` and `--session/-s <id>`, plus `--fork`
  - `--title`
  - `--attach http://host:4096` sends the run to an already running server.
  - `--username/-u` and `--password/-p` for basic auth
  - `--dir`, `--port`
  - `--variant` sets reasoning effort.
  - `--thinking`
  - `--command`
  - `--auto` means "Auto-approve permissions that are not explicitly denied."
  - Global flags: `--pure` runs without external plugins, and `--print-logs` writes logs to stderr.
  - Source: [OpenCode CLI docs](https://opencode.ai/docs/cli/)
- The docs recommend `opencode serve` plus `opencode run --attach http://localhost:4096 "..."` "to avoid MCP server cold boot on every run". This is directly relevant to frequent polling. — [OpenCode CLI docs](https://opencode.ai/docs/cli/)
- Relevant environment variables:
  - `OPENCODE_PERMISSION`: an inline JSON permission config
  - `OPENCODE_CONFIG`: path to the config file
  - `OPENCODE_CONFIG_CONTENT`: inline JSON config
  - `OPENCODE_CONFIG_DIR`
  - `OPENCODE_DISABLE_AUTOUPDATE`
  - `OPENCODE_DISABLE_MODELS_FETCH` and `OPENCODE_MODELS_URL`, useful for air-gapped hosts
  - `OPENCODE_DISABLE_DEFAULT_PLUGINS`
  - `OPENCODE_DISABLE_AUTOCOMPACT`
  - `OPENCODE_SERVER_PASSWORD` and `OPENCODE_SERVER_USERNAME`
  - Source: [OpenCode CLI docs](https://opencode.ai/docs/cli/)
- **Exit codes are not documented** in the CLI docs. — [OpenCode CLI docs](https://opencode.ai/docs/cli/)
- **Reliability caveat:** issue anomalyco/opencode#38723 reports `opencode run` hanging intermittently before it creates a session. The process stays alive and prints no output. The reporter measured 27/61 successes (44%) over one day. The reporter warns that stdout is buffered until completion, so a hung run and a slow run look the same from the shell, and suggests checking the log for session creation instead. — [anomalyco/opencode#38723](https://github.com/anomalyco/opencode/issues/38723) (single user report; frequency in other environments unknown)
- A third-party project (codeframe PR #1008) found that its adapter had passed a nonexistent `--non-interactive` flag, so the delegated run did no work. The PR also notes that `--auto` is opt-in because it is dangerous. — [frankbria/codeframe PR #1008](https://github.com/frankbria/codeframe/pull/1008)
- Older documentation for the Go-based OpenCode said non-interactive mode auto-approved all permissions. That behaviour is from a superseded version, so do not rely on it. — [old mirrored commit](https://git.joshthomas.dev/mirrors/opencode/commit/623d132772b9c69dd6d99ed4004b26c46dbe43a4)

**`opencode serve` (HTTP server)**
- `opencode serve [--port] [--hostname] [--cors] [--mdns]` defaults to port 4096 on 127.0.0.1. Setting `OPENCODE_SERVER_PASSWORD` enables HTTP basic auth, with username `opencode` by default. — [OpenCode server docs](https://opencode.ai/docs/server/)
- Key endpoints:
  - `POST /session` creates a session.
  - `POST /session/:id/message` sends a prompt and waits for the reply. The body takes `parts`, `model`, `agent`, `system`, `tools` and `noReply`.
  - `POST /session/:id/prompt_async` sends a prompt and returns 204.
  - `GET /event` is an SSE event stream.
  - `POST /session/:id/permissions/:permissionID` with `{response, remember?}` answers a permission prompt programmatically.
  - `GET /doc` serves the OpenAPI spec.
  - Source: [OpenCode server docs](https://opencode.ai/docs/server/)

**SDK (`@opencode-ai/sdk`)**
- Install with `npm install @opencode-ai/sdk`.
  - `createOpencode({hostname, port, config})` starts a server and returns a client. The inline `config` overrides `opencode.json`.
  - `createOpencodeClient({baseUrl})` connects to a server that is already running.
  - Source: [OpenCode SDK docs](https://opencode.ai/docs/sdk/)
- Structured output: `session.prompt({ body: { parts, format: { type: "json_schema", schema, retryCount } } })` returns validated JSON in `result.data.info.structured_output`. If validation still fails after the retries, the result carries `StructuredOutputError`. The docs' table names the field `outputFormat` while the examples use `format`, so verify against your server version. — [OpenCode SDK docs](https://opencode.ai/docs/sdk/)

```ts
// monitor-analyze.ts: called only when the deterministic poller detects an anomaly
import { createOpencodeClient } from "@opencode-ai/sdk"
const client = createOpencodeClient({ baseUrl: "http://127.0.0.1:4096" })
const session = await client.session.create({ body: { title: `RMF anomaly ${new Date().toISOString()}` } })
const res = await client.session.prompt({
  path: { id: session.data!.id },           // check the response shape for your SDK version
  body: {
    parts: [{ type: "text", text: `Analyze this RMF III/OMEGAMON snapshot and classify severity:\n${snapshotJson}` }],
    format: { type: "json_schema", schema: {
      type: "object",
      properties: {
        severity: { type: "string", enum: ["info","warn","critical"] },
        summary: { type: "string" },
        likely_cause: { type: "string" },
        notify: { type: "boolean" }
      },
      required: ["severity","summary","notify"] } }
  }
})
const verdict = res.data.info.structured_output  // drive notification from this, deterministically
```

**Permissions for unattended runs**
- `permission` can be a single value (`"allow"`) or an object keyed by tool. The values are `allow`, `ask` and `deny`. Keys:
  - `read`, `edit` (covers write/patch), `glob`, `grep`, `bash`, `task`, `skill`, `lsp`
  - `question`, `webfetch`, `websearch`
  - `external_directory`, `doom_loop`
  - Granular rules use `*` and `?` globs, and the **last matching rule wins**, so put `"*"` first.
  - Source: [OpenCode permissions docs](https://opencode.ai/docs/permissions/)
- Defaults: most permissions are `allow`, while `doom_loop` and `external_directory` default to `ask`. `read` denies `*.env` and `*.env.*` by default. — [OpenCode permissions docs](https://opencode.ai/docs/permissions/)
- The docs do not say what happens to an `ask` rule under `opencode run` with no TTY. `--auto` approves anything not explicitly denied, and "Explicit `deny` rules are still enforced in auto mode." — [OpenCode permissions docs](https://opencode.ai/docs/permissions/)
- Per-agent `permission` blocks merge with the global config, and the agent's rules take precedence. — [OpenCode permissions docs](https://opencode.ai/docs/permissions/)

Example of a locked-down monitoring config. It uses only documented keys, but the combination itself is untested.
```json
{
  "$schema": "https://opencode.ai/config.json",
  "autoupdate": false,
  "permission": {
    "*": "deny",
    "read": "allow",
    "grep": "allow",
    "glob": "allow",
    "edit": "deny",
    "bash": { "*": "deny", "python3 /opt/mon/bin/query_rmf.py *": "allow" },
    "webfetch": { "*": "deny", "https://zosmf.example.internal/*": "allow" },
    "websearch": "deny",
    "question": "deny",
    "doom_loop": "deny",
    "external_directory": "deny"
  }
}
```

### Inferences
- For 1–15 minute polling, run a long-lived `opencode serve` under systemd and invoke it via the SDK or `run --attach`. This avoids cold-starting MCP servers on every tick, the cost the docs warn about.
- Because exit codes are undocumented and runs can hang (#38723), wrap every invocation in `timeout 300 opencode run ...`. Judge success by parsing the JSON events or the structured output, not by the exit status.
- Set every key the agent touches to explicit `allow` or `deny`. Avoid `ask` in unattended mode, and prefer explicit allowlists to `--auto`, so that a prompt injected through monitored data cannot widen what the agent may do.

### Gaps
- Exit-code semantics of `opencode run` (success, model error, permission denied) are undocumented. They would need empirical testing or a read of the source.
- The JSON event schema emitted by `run --format json` was not documented on the pages fetched.
- Behaviour of `ask` permissions under `run` without `--auto` (blocks, fails or auto-rejects) is not documented.

## 2. Custom tools, MCP servers, agents, plugins and rules (AGENTS.md)

### Takeaway
OpenCode can be extended in five ways:
- TypeScript tools in `.opencode/tools/`, which can shell out to Python or REXX wrappers
- local or remote MCP servers in `opencode.json`
- markdown agents in `.opencode/agents/`, with their own model, step limit and permission set
- JS/TS plugins with hooks such as `tool.execute.before` (for blocking or auditing) and `session.idle` (for notification)
- `AGENTS.md` and `instructions` files for standing rules such as threshold policy and runbooks

### Cited Findings
**Custom tools**
- Tools live in `.opencode/tools/` (project) or `~/.config/opencode/tools/` (global), and the filename becomes the tool name. Use the `tool()` helper from `@opencode-ai/plugin`, which takes `args` as Zod via `tool.schema`. `execute(args, context)` receives `agent`, `sessionID`, `messageID`, `directory` and `worktree`. Each named export in a file becomes a separate tool called `<file>_<export>`. — [OpenCode custom tools docs](https://opencode.ai/docs/custom-tools/)
- Tools can call other languages through `Bun.$`, for example ``await Bun.$`python3 ${script} ${args.a}`.text()``. A custom tool with the same name as a built-in tool overrides it. — [OpenCode custom tools docs](https://opencode.ai/docs/custom-tools/)

```ts
// .opencode/tools/rmf.ts  -> tools rmf_cpu, rmf_delays (illustrative; endpoint path is a placeholder)
import { tool } from "@opencode-ai/plugin"
export const cpu = tool({
  description: "Fetch RMF Monitor III CPC/CPU utilization for an LPAR (read-only)",
  args: { lpar: tool.schema.string().describe("LPAR/system name, e.g. SYSA") },
  async execute(args) {
    return await Bun.$`python3 /opt/mon/bin/query_rmf.py cpu ${args.lpar}`.text()
  },
})
export const delays = tool({
  description: "Fetch RMF III job delay report (DELAY) for an LPAR",
  args: { lpar: tool.schema.string() },
  async execute(args) {
    return await Bun.$`python3 /opt/mon/bin/query_rmf.py delay ${args.lpar}`.text()
  },
})
```

**MCP servers**
- A local server uses `"type": "local"` with `command` (an array), `environment`, `cwd`, `enabled` and `timeout` (default 5000 ms). A remote server uses `"type": "remote"` with `url`, `headers` and optional `oauth` (clientId/clientSecret/scope, or `false` for API-key servers). Authenticate with `opencode mcp auth <name>` and check status with `opencode mcp list`. Values support `{env:VAR}` substitution. — [OpenCode MCP docs](https://opencode.ai/docs/mcp-servers/)
- MCP tools are prefixed with the server name. You can disable them globally (`"tools": {"zowe*": false}`) and re-enable them for a specific agent (`"agent": {"zos-monitor": {"tools": {"zowe*": true}}}`). — [OpenCode MCP docs](https://opencode.ai/docs/mcp-servers/)
- The docs warn that MCP servers add to context and that this "can quickly add up". Large servers such as GitHub's can exceed the context limit. — [OpenCode MCP docs](https://opencode.ai/docs/mcp-servers/)

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "zowe": {
      "type": "local",
      "command": ["node", "/opt/zowe-mcp/dist/index.js"],
      "environment": { "ZOWE_MCP_TIER": "read" },
      "timeout": 15000
    },
    "mailer": {
      "type": "local",
      "command": ["npx", "-y", "<smtp-mcp-package>"],
      "environment": { "SMTP_HOST": "{env:SMTP_HOST}", "SMTP_USER": "{env:SMTP_USER}", "SMTP_PASS": "{env:SMTP_PASS}" }
    }
  },
  "tools": { "zowe*": false, "mailer*": false },
  "agent": { "zos-monitor": { "tools": { "zowe*": true, "mailer*": true } } }
}
```
(The flag and env names for Zowe MCP above are illustrative. See section 6 for the server's real configuration surface.)

**Agents**
- Agents are markdown files in `.opencode/agents/*.md` or `~/.config/opencode/agents/*.md`, and the filename becomes the agent name. Frontmatter fields:
  - `description` (required)
  - `mode` (`primary`/`subagent`/`all`)
  - `model` (`provider/model-id`)
  - `temperature`, `top_p`
  - `steps`, the maximum agentic iterations (`maxSteps` is deprecated)
  - `permission` (`tools` is deprecated)
  - `disable`, `hidden`, `color`
  - The markdown body is the system prompt. JSON agents use `prompt: "{file:...}"`.
  - Built-in agents: `build`, `plan`, `general`, `explore`, `scout`.
  - Source: [OpenCode agents docs](https://opencode.ai/docs/agents/)

```markdown
---
description: Analyzes z/OS performance anomalies from RMF III / OMEGAMON data and drafts alerts
mode: primary
model: anthropic/claude-sonnet-4-5
temperature: 0.1
steps: 12
permission:
  edit: deny
  bash:
    "*": deny
    "python3 /opt/mon/bin/query_rmf.py *": allow
  webfetch: deny
  websearch: deny
  question: deny
---
You are a z/OS performance analyst. Use only rmf_* and zowe_* read tools.
Never issue operator commands. Output must follow the JSON schema provided.
```

**Plugins**
- Plugins load from `.opencode/plugins/`, `~/.config/opencode/plugins/`, or npm packages listed under `"plugin"`. A plugin is `async ({project, client, $, directory, worktree}) => ({...hooks})`. Events include:
  - `tool.execute.before` and `tool.execute.after`
  - `session.idle`, `session.error`, `session.created`
  - `permission.asked` and `permission.replied`
  - `message.updated`, `file.edited`, `shell.env`
  - Throwing inside `tool.execute.before` blocks the call.
  - Source: [OpenCode plugins docs](https://opencode.ai/docs/plugins/)

```js
// .opencode/plugins/audit.js: audit log and block anything that looks like an operator/command tool
export const Audit = async ({ $ }) => ({
  "tool.execute.before": async (input, output) => {
    if (/console|tso|submit|cancel|delete/i.test(input.tool)) throw new Error(`blocked tool ${input.tool}`)
    await $`logger -t opencode-audit ${JSON.stringify({ tool: input.tool, args: output.args })}`
  },
  event: async ({ event }) => {
    if (event.type === "session.error") await $`logger -t opencode-audit session.error`
  },
})
```
(Pattern adapted from the docs' `.env` protection and notification examples. The regex and the use of `logger` are this note's own illustration.)

**Rules**
- `AGENTS.md` in the project root (walked up from the current directory) and `~/.config/opencode/AGENTS.md` are loaded into context. `CLAUDE.md` is a fallback, which `OPENCODE_DISABLE_CLAUDE_CODE=1` turns off. The `instructions` array in `opencode.json` accepts paths, globs and remote URLs (fetched with a 5 s timeout), for example `["runbooks/*.md"]`. — [OpenCode rules docs](https://opencode.ai/docs/rules/)

### Inferences
- Keep thresholds, LPAR names, WLM service-class goals and escalation rules in `AGENTS.md` or `instructions` files (version-controlled runbooks). Keep data access in narrowly scoped custom tools that wrap existing, already authorised APIs such as z/OSMF REST, the RMF DDS (Distributed Data Server) XML/HTTP API and OMEGAMON data providers.
- Custom tools are often safer than generic MCP servers here. They expose only the exact read calls needed and add few tokens to context.

### Gaps
- The docs do not say whether `tool.execute.before` sees MCP tool calls as well as built-in ones. The plugin list implies it does, but this is unverified.
- No first-party OMEGAMON or RMF DDS MCP server or OpenCode tool was found (see section 6).

## 3. Model provider configuration (Anthropic, OpenAI, local or Ollama for data-sensitive mainframe shops)

### Takeaway
OpenCode is provider-agnostic. It supports Anthropic, OpenAI, Azure OpenAI, Amazon Bedrock and any OpenAI-compatible local endpoint (Ollama, LM Studio). `enabled_providers` acts as an allowlist, so you can guarantee data never reaches a public API.

### Cited Findings
- **Ollama:** set `provider.ollama` with `npm: "@ai-sdk/openai-compatible"`, `options.baseURL: "http://localhost:11434/v1"` and a `models` map. The docs suggest raising Ollama `num_ctx` to roughly 16k–32k if tool calls fail. LM Studio follows the same pattern at `http://127.0.0.1:1234/v1`. — [OpenCode providers docs](https://opencode.ai/docs/providers/)
- **Bedrock:** set `provider["amazon-bedrock"].options` with `region` and `profile`. `AWS_BEARER_TOKEN_BEDROCK` overrides the profile. **Azure OpenAI:** use the `AZURE_RESOURCE_NAME` env var and map catalog models to deployment names via `models.<name>.id`. — [OpenCode providers docs](https://opencode.ai/docs/providers/)
- **Credentials:**
  - Keys entered with `/connect` are stored in `~/.local/share/opencode/auth.json`, and `opencode auth list` shows which are configured.
  - Config files can reference secrets with `{env:ANTHROPIC_API_KEY}` or `{file:~/.secrets/key}`. An unset env var becomes an empty string.
  - Sources: [OpenCode providers docs](https://opencode.ai/docs/providers/); [OpenCode config docs](https://opencode.ai/docs/config/)
- **Restricting providers:**
  - `model` and `small_model` set the main model and the one used for titles and light tasks.
  - `enabled_providers` is an allowlist and `disabled_providers` a blocklist. `disabled_providers` wins if a provider appears in both.
  - Source: [OpenCode config docs](https://opencode.ai/docs/config/)
- **Config precedence (later overrides earlier, all merged):**
  1. remote `.well-known/opencode`
  2. global `~/.config/opencode/opencode.json`
  3. `OPENCODE_CONFIG`
  4. project `opencode.json`
  5. `.opencode` directories
  6. `OPENCODE_CONFIG_CONTENT`
  7. managed config files
  8. macOS managed preferences
  - Source: [OpenCode config docs](https://opencode.ai/docs/config/)

```json
{
  "$schema": "https://opencode.ai/config.json",
  "enabled_providers": ["ollama"],
  "model": "ollama/qwen2.5-coder:32b",
  "small_model": "ollama/qwen2.5-coder:32b",
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Ollama (on-prem)",
      "options": { "baseURL": "http://llm-gw.internal:11434/v1" },
      "models": { "qwen2.5-coder:32b": { "name": "Qwen2.5 Coder 32B" } }
    }
  }
}
```

### Inferences
- For mainframe performance data, which can include job names, user IDs and dataset names that may be sensitive, use `enabled_providers` with an on-prem model, or a cloud model under an enterprise agreement (Bedrock or Azure in the shop's own tenant). Also set `small_model` to the same provider, otherwise title generation could send prompt fragments to a different provider. The GitLab example in the docs does this for that reason.
- Set `OPENCODE_DISABLE_MODELS_FETCH` (or point `OPENCODE_MODELS_URL` at an internal mirror) and `autoupdate: false` on air-gapped or locked-down hosts. These variables are documented in the CLI docs. Their use for this purpose is an inference.
- Local models tend to be weaker at multi-step tool calling. Keep the LLM's job small (classify and summarise a pre-collected snapshot), which suits the anomaly-only pattern described in section 4.

### Gaps
- No benchmark was found for which local models handle OpenCode tool calling reliably.
- The pages fetched did not document whether the session "share" feature can be fully disabled by config. Verify this for compliance.

## 4. Scheduling (cron, systemd, Kubernetes, GitHub Actions) and the "deterministic poller, LLM only on anomalies" pattern

### Takeaway
OpenCode has no built-in scheduler. The only first-party scheduling integration is the GitHub Action with `on: schedule`. For on-prem monitoring, use cron or systemd timers (or a Kubernetes CronJob) to run a deterministic poller. Invoke OpenCode, preferably through a persistent `opencode serve`, only when thresholds are breached.

### Cited Findings
- The OpenCode GitHub Action (`anomalyco/opencode/github@latest`) supports `schedule` (cron) and `workflow_dispatch` triggers. Both require a `prompt` input. Other inputs are `model` (required), `agent`, `variant` and `use_github_token`. Scheduled output goes to logs and PRs, not to issue comments. — [OpenCode GitHub docs](https://opencode.ai/docs/github/)

```yaml
on:
  schedule: [{ cron: "*/30 * * * *" }]
  workflow_dispatch:
jobs:
  opencode:
    runs-on: ubuntu-latest
    permissions: { id-token: write, contents: read }
    steps:
      - uses: actions/checkout@v6
        with: { persist-credentials: false }
      - uses: anomalyco/opencode/github@latest
        env: { ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }} }
        with:
          model: anthropic/claude-sonnet-4-20250514
          agent: build
          prompt: "Summarize ..."
```
(This is adapted from the docs' scheduled example. GitHub-hosted runners usually cannot reach internal z/OS endpoints, so a self-hosted runner would be needed. That last point is an inference.)
- The docs recommend a persistent `opencode serve` with `run --attach` to avoid cold-booting MCP servers on each run. — [OpenCode CLI docs](https://opencode.ai/docs/cli/)
- `opencode run` can hang intermittently before it creates a session (issue #38723). An external timeout and log-based health checks are therefore needed. — [anomalyco/opencode#38723](https://github.com/anomalyco/opencode/issues/38723)

Reference implementation (the author's own design, combining the documented flags above):
```ini
# /etc/systemd/system/opencode-serve.service
[Service]
User=zmon
EnvironmentFile=/etc/zmon/secrets.env          # ANTHROPIC_API_KEY / OPENCODE_SERVER_PASSWORD etc. (mode 0600)
Environment=OPENCODE_DISABLE_AUTOUPDATE=1
WorkingDirectory=/opt/zmon                      # contains opencode.json, AGENTS.md, .opencode/
ExecStart=/usr/local/bin/opencode serve --hostname 127.0.0.1 --port 4096
Restart=always

# /etc/systemd/system/zmon-poll.timer -> OnCalendar=*:0/5   (every 5 min)
# /etc/systemd/system/zmon-poll.service -> ExecStart=/opt/zmon/bin/poll.sh
```
```bash
#!/usr/bin/env bash
# /opt/zmon/bin/poll.sh: deterministic polling and thresholding; LLM only on breach
set -euo pipefail
snap=$(python3 /opt/zmon/bin/collect.py)            # RMF DDS / z/OSMF / OMEGAMON REST -> JSON
breach=$(echo "$snap" | jq '[.metrics[] | select(.value > .threshold)] | length')
[ "$breach" -eq 0 ] && exit 0                         # no anomaly: no tokens spent
# dedupe / cooldown so the same breach doesn't call the LLM every tick
key=$(echo "$snap" | jq -r '[.metrics[]|select(.value>.threshold)|.id]|join(",")')
[ -f "/var/lib/zmon/cool/$key" ] && find "/var/lib/zmon/cool/$key" -mmin -30 | grep -q . && exit 0
touch "/var/lib/zmon/cool/$key"
timeout 300 opencode run --attach http://127.0.0.1:4096 --agent zos-monitor --format json \
  -f <(echo "$snap") "Analyze the attached snapshot; follow AGENTS.md escalation policy." \
  > "/var/log/zmon/run-$(date +%s).jsonl" \
  || /opt/zmon/bin/notify_fallback.sh "$snap"       # deterministic alert if LLM fails/hangs
```
A Kubernetes CronJob follows the same shape (`schedule: "*/5 * * * *"`, `concurrencyPolicy: Forbid`, `activeDeadlineSeconds: 300`), with secrets mounted from a Secret or Vault. That is standard Kubernetes and is not OpenCode-specific.

### Inferences
- **Cost and reliability.** At a 5-minute interval, invoking the LLM unconditionally means 288 runs per day per check. Gating on deterministic thresholds keeps model calls proportional to incidents, makes alerting work even when the LLM is down or hangs, and keeps the system of record auditable.
- **Division of labour.** The LLM adds value in correlating signals (for example CPU and delay reports, WLM goals, recent job submissions), writing a human-readable summary, and suggesting a likely cause. The *decision* to page should stay deterministic, or at least be validated by schema (the SDK's `format: json_schema`).
- **Hangs.** A fallback non-LLM notification path is essential, given the reported hangs and undocumented exit codes.
- **Overlap.** Use `concurrencyPolicy: Forbid`, `flock`, or the systemd single-instance behaviour to prevent overlapping runs.

### Gaps
- No first-party OpenCode documentation or tutorial was found on cron, systemd or Kubernetes scheduling, or on built-in recurring tasks. The patterns above are synthesised.
- No published cost figures for OpenCode-based monitoring were found.

## 5. Security: secrets, tool restriction, logging and audit

### Takeaway
OpenCode provides the building blocks but no turnkey audit:
- `{env:}` and `{file:}` secret substitution
- allow/deny permission globs, with per-agent overrides that `--auto` cannot bypass
- `.env` read-deny defaults
- plugin hooks (`tool.execute.before`/`after`, `permission.*`, `session.error`) that can enforce and log policy
- basic auth on `serve`

Audit logging and data-trust boundaries must be built by the operator.

### Cited Findings
- **Secrets:** use `{env:VAR}` or `{file:path}` in config. Provider keys stored via `/connect` sit in `~/.local/share/opencode/auth.json`. — [OpenCode config docs](https://opencode.ai/docs/config/); [OpenCode providers docs](https://opencode.ai/docs/providers/)
- **Default protections:** reads of `*.env` and `*.env.*` are denied by default. `external_directory` and `doom_loop` default to `ask`. Explicit `deny` is enforced even with `--auto`. — [OpenCode permissions docs](https://opencode.ai/docs/permissions/)
- **Blocking:** a plugin that throws inside `tool.execute.before` blocks the tool call. `permission.asked`, `permission.replied` and `session.error` events are available for logging. — [OpenCode plugins docs](https://opencode.ai/docs/plugins/)
- **Server auth:** `opencode serve` binds to 127.0.0.1 by default, and `OPENCODE_SERVER_PASSWORD` enables basic auth. — [OpenCode server docs](https://opencode.ai/docs/server/)
- `--print-logs` writes logs to stderr, and `--pure` disables external plugins. — [OpenCode CLI docs](https://opencode.ai/docs/cli/)
- **MCP context cost:** unneeded MCP servers or tools can be disabled globally and enabled per agent with name globs. — [OpenCode MCP docs](https://opencode.ai/docs/mcp-servers/)
- The Zowe MCP README states that "only z/OS access controls actually enforce boundaries". Its command gates and capability tiers are defence in depth, not a security boundary. — [zowe/zowe-mcp](https://github.com/zowe/zowe-mcp)

### Inferences
- Run under a dedicated non-root service user. Give it a RACF/ACF2/Top Secret ID with read-only access to the RMF and OMEGAMON data it needs and no operator-command authority. That keeps the mainframe's own access control as the hard boundary.
- Treat monitored data (job names, messages, SYSLOG text) as untrusted input that could carry prompt-injection payloads. That is the reason for deny-by-default permissions, `webfetch` allowlists, and no write or command tools for the monitoring agent.
- Keep `--format json` event logs and plugin audit records as the audit trail, and ship them to SIEM (for example via syslog, or SMF on the z/OS side for API calls).
- Do not use `--auto` in production. Use an explicit allowlist config so every grant is reviewable in version control.

### Gaps
- No documentation was found of a built-in persistent audit log, of OpenCode telemetry or data collection, or of where session transcripts are stored on disk.
- The pages fetched did not say whether `OPENCODE_PERMISSION` overrides or merges with file-based permission config.

## 6. Existing MCP servers: email (SMTP), Teams/Graph, HTTP/fetch, Zowe/z/OS

### Takeaway
The best z/OS fit is `zowe/zowe-mcp`, which offers data sets, USS, jobs and TSO/console commands over SSH with read-only tiers. It is early-stage and not yet on npm. Several community servers exist for SMTP email. Teams is the hardest channel: the official Microsoft server (preview) and Graph message-posting require delegated user auth, which suits headless agents poorly. No OMEGAMON- or RMF-specific MCP server was found.

### Cited Findings
**z/OS**
- **zowe/zowe-mcp** (EPL-2.0, in the `zowe` GitHub org):
  - Tools: data sets (list, read, write, create, delete, copy, rename, search, attributes); USS files; TSO and console commands; jobs (submit, status, list, output, cancel, hold, release, delete); `getContext` and `setSystem`.
  - Backend: SSH. Auth order is SSH key, then password env var, then Vault KV, then an interactive prompt. ssh-agent is not supported. z/OSMF is not mentioned.
  - Capability tiers: `read-strict` (the default: read-only with confirmation), `read` (read-only, auto-approved), `update`, `delete` and `full`. A mock mode exists.
  - `@zowe/mcp-server` is **not yet published on npm** (the npx command returns 404). Build from source or install a CI tarball.
  - Maturity signals: about 281 commits, 20 stars, a nightly SDK default, and an evals suite with a safety guide.
  - Source: [zowe/zowe-mcp](https://github.com/zowe/zowe-mcp)
- **Zowe CLI MCP** (community) exposes only `get-user-jobs` and `get-jobs-by-name`, via z/OSMF profiles. — [mcp.so Zowe CLI MCP](https://mcp.so/servers/zowe-cli-mcp) (aggregator listing)
- **NotHarshhaa/mainframe-mcp-server** (community) covers jobs, datasets, USS and diagnostics via z/OSMF. Install with `npx -y @notharshhaa/mainframe-mcp-server`, with host and credentials in env vars. — [claudemarket listing](https://claudemarket.ai/mcp/NotHarshhaa/mainframe-mcp-server) (aggregator; unverified against the repo)

**Email (SMTP)**
- Community options (from directory listings):
  - fahadBSTech: a minimal nodemailer server with one `send_email` tool, configured through `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER` and `SMTP_PASS`
  - maxyychen/mcp-sendmail: Streamable HTTP, TLS, templates, Docker
  - samihalawa/mcp-server-smtp: multiple SMTP configs, rate-limited bulk sending
  - bedro96/email-send-mcp: FastMCP, SMTP plus IMAP/POP3
  - yunfeizhu/mcp-mail-server
  - Sources: [Glama fahadBSTech](https://glama.ai/mcp/servers/@fahadBSTech/mcp-gmail-sender/blob/7bb2fcf881bd11126559fb53ccd5e0155961818e/README.md); [Glama mcp-sendmail](https://glama.ai/mcp/servers/@maxyychen/mcp-sendmail/blob/2eb9a21cb0bc46235fe10fd31d979daaa4d47f1c/README.md); [Glama mcp-server-smtp](https://glama.ai/mcp/servers/@samihalawa/mcp-server-smtp/blob/fef016419d83069c55dd83fadbae996ff4af0f9c/README.md); [Glama email-send-mcp](https://glama.ai/mcp/servers/@bedro96/email-send-mcp/tree/8b623c859a49bb79791b8ca9c3e832c138cd45a6). Maintenance and licensing were not verified.

**Microsoft Teams / Graph**
- According to a Scalekit blog post, Microsoft's official "Work IQ Teams" MCP server is in preview with about two dozen Graph-backed tools. It uses Entra ID **delegated permissions only**, so app-only (headless) auth is not supported. Graph restricts app-only message posting to migration scenarios (`Teamwork.Migrate.All`). — [Scalekit blog](https://www.scalekit.com/blog/microsoft-teams-mcp-vs-api) (secondary source; not confirmed on Microsoft docs)
- Community servers:
  - floriscornel/teams-mcp offers `send_chat_message` and a read-only mode.
  - ssasuoirafen/teams-mcp-server needs an Azure AD app registration with delegated Graph permissions.
  - imatiach-msft/teams-mcp offers a Power Automate route.
  - Sources: [mcpservers.org floriscornel/teams-mcp](https://mcpservers.org/servers/floriscornel/teams-mcp); [Glama ssasuoirafen](https://glama.ai/mcp/servers/ssasuoirafen/teams-mcp-server); [remoteopenclaw imatiach-msft](https://www.remoteopenclaw.com/mcp/imatiach-msft/teams-mcp)

**Generic HTTP**
- OpenCode has a built-in `webfetch` tool whose permission can be scoped to URL patterns, for example only `https://zosmf.internal/*`. — [OpenCode permissions docs](https://opencode.ai/docs/permissions/)

### Inferences
- For unattended Teams alerts, a deterministic POST to a Teams incoming webhook or a Power Automate "workflow" URL, made from the poller script or a tiny custom tool, is simpler and more reliable than a Graph MCP server that needs delegated user tokens. The same applies to email: a 10-line custom tool or script around the shop's SMTP relay avoids a third-party dependency.
- Configure zowe-mcp at the `read` tier (no write or console) for the monitoring agent, and remove command tools entirely. Note that zowe-mcp works over SSH and USS. RMF Monitor III data is usually reached through the RMF Distributed Data Server (DDS) HTTP API or the z/OSMF RMF interfaces, which zowe-mcp does not appear to expose. A small custom tool wrapping DDS is likely needed.

### Gaps
- No IBM-published z/OS MCP server (for example from IBM Z or z/OS Connect) was found in the searches run. Check IBM sources separately.
- No MCP server for OMEGAMON (Data Provider / TEMS REST) or RMF DDS was found.
- The Microsoft Work IQ Teams server was not confirmed on a Microsoft-owned page.

## 7. Disambiguation: other "open code AI agents" and how they differ

### Takeaway
"OpenCode" (opencode.ai, Anomaly/SST) is a terminal-first, provider-agnostic coding agent with a client/server architecture. It should not be confused with OpenHands (formerly OpenDevin) or with older or forked "opencode" projects. OpenHands also offers a headless CLI with JSONL output, but in headless mode it always auto-approves every action, which matters for unattended safety.

### Cited Findings
- **OpenHands:**
  - Headless CLI mode, with the task given via `-t` or `-f`
  - "Always runs in always-approve mode", so the agent executes every action without confirmation, and `--llm-approve` is unavailable
  - `--json` produces JSONL event output
  - Installed with `uv tool install openhands --python 3.12`
  - Sources: [OpenHands headless docs](https://docs.openhands.dev/openhands/usage/cli/headless); [PyPI openhands](https://www.pypi.org/project/openhands/)
- **OpenHands configuration gotcha:** without `--override-with-envs`, OpenHands uses `~/.openhands/settings.json` and ignores `LLM_*` env vars, which matters for cron. — [Hermes Agent skill page](https://hermes-agent.nousresearch.com/docs/user-guide/skills/optional/autonomous-ai-agents/autonomous-ai-agents-openhands) (third-party, checked against CLI 1.16.0)
- **Legacy Go-based OpenCode:** an earlier opencode codebase described a non-interactive mode in which all permissions were auto-approved. Its docs survive in mirrors and should not be confused with the current TypeScript/Bun OpenCode. — [old mirrored commit](https://git.joshthomas.dev/mirrors/opencode/commit/623d132772b9c69dd6d99ed4004b26c46dbe43a4)

### Inferences
- For a monitoring use case, OpenCode's granular allow/deny permissions, persistent server and SDK structured output make it better suited than OpenHands' always-approve headless mode. OpenHands is oriented to sandboxed (Docker) software-engineering tasks.
- Both are general coding agents repurposed for monitoring. A thin script plus a direct LLM API call (or an agent framework such as the Claude Agent SDK, LangGraph and similar) may be simpler when the LLM's role is only "classify and summarise this snapshot". Those alternatives were not researched in depth here.

### Gaps
- Comparable alternatives (Goose, Aider, the Claude Agent SDK, n8n with an LLM node) were not researched in this pass.
- No authoritative source was found explaining the sst to anomalyco repository move or the history of the Go fork.
