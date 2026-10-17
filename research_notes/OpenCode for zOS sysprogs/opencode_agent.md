# OpenCode AI coding agent: what it is and how to install, configure and extend it (for z/OS sysprogs, as of Oct 2026)

Research date: 2026-10-17. OpenCode currently has **two parallel major lines: V1 (1.19.x, docs at opencode.ai/docs) and V2 (2.0.x, docs at opencode.ai/v2/docs)**. Most config details below are V1 unless marked V2. Their config schemas differ, so check which one is installed before following any guide.

## 1. What OpenCode is, who maintains it, license, version and status in 2026

### Takeaway
OpenCode is an MIT-licensed, open-source AI coding agent with terminal (TUI), desktop and web/IDE front ends. It is now hosted at github.com/anomalyco/opencode (formerly sst/opencode) and run by the company Anomaly. As of mid-October 2026, V1 (1.19.3) is the default install and V2 (2.0.x, released 11 Sep 2026) is the opt-in new architecture.

### Cited Findings
- OpenCode calls itself "an open source AI coding agent" and ships as a terminal interface, desktop app, or IDE extension — [OpenCode docs](https://opencode.ai/docs)
- The repo is `anomalyco/opencode`, MIT license, about 215.2k stars, 28.9k forks (at time of fetch). It describes itself as "The open source coding agent". The GitHub page names no individual maintainers — [GitHub repo](https://github.com/sst/opencode) (the sst/opencode URL serves anomalyco/opencode)
- Docker images, Homebrew tap, mise and GitHub Action references now all use the `anomalyco` namespace (`ghcr.io/anomalyco/opencode`, `brew install anomalyco/tap/opencode`, `anomalyco/opencode/github@latest`) — [OpenCode docs](https://opencode.ai/docs), [GitHub integration docs](https://opencode.ai/docs/github/)
- "Anomaly" appears to be the company behind OpenCode. WinGet ID for V2 is `Anomaly.OpenCode` — [capitalandcompute.net](https://capitalandcompute.net/blog/opencode-2-0-what-changed/)
- Latest V1 releases: v1.19.3 (16 Oct 2026), v1.19.2 (16 Oct), v1.19.1 (15 Oct). Releases are very frequent (several per week) — [GitHub releases](https://github.com/anomalyco/opencode/releases)
- v1.19.0 (11 Oct 2026) added `opencode upgrade --major` to move to V2 and stopped auto-installing updates. v1.19.2 pins V1 users to 1.x: update checks never offer 2.x, and the curl install script refuses to install 2.x. Homebrew Core's `opencode` formula now tracks V2 — [GitHub releases](https://github.com/anomalyco/opencode/releases)
- V2: npm `@opencode/cli` 2.0.0 was published 11 Sep 2026 and reached 2.0.28 by 13 Oct 2026. The older `opencode-ai` npm package still tags 1.x as `latest`. No source labels V2 explicitly "stable" — [capitalandcompute.net](https://capitalandcompute.net/blog/opencode-2-0-what-changed/); V2 docs mention an AUR package named `opencode-beta` and Homebrew `opencode-v2` and do not state a stability level — [V2 docs](https://opencode.ai/v2/docs/)
- V2 architecture: one shared background service that the terminal, desktop and web apps all connect to. Registers at `~/.local/state/opencode/service.json`, sessions in `~/.local/share/opencode/opencode.db`, `opencode service status`. New features include "Code Mode" (a sandboxed JS `execute` tool), background subagents, worktree tools, and `opencode pair` for web access — [capitalandcompute.net](https://capitalandcompute.net/blog/opencode-2-0-what-changed/) (cites The New Stack, 12 Oct 2026)
- V1 and V2 both use the `opencode` command and cannot be installed side by side under the default name. V1 install methods can replace a V2 install — [OpenCode docs](https://opencode.ai/docs)
- Hosted offerings: "OpenCode Zen" (curated, tested model gateway, recommended for new users), the OpenCode Console, and "OpenCode Go" ($10/month plan for open-source models) — [OpenCode docs](https://opencode.ai/docs), [V2 docs](https://opencode.ai/v2/docs/)

### Inferences
- A site standardising on OpenCode should pin a major version (V1 1.19.x or V2 2.0.x). Config keys, plugin APIs, the server API and LSP behaviour all differ between them (see section 4).
- The project releases multiple versions a week. Regulated sites will want `autoupdate: false` and an internal mirror or approved-version process.

### Gaps
- Could not query the GitHub API directly (session restriction), so exact contributor and maintainer names and the full asset list were not retrieved.
- No official statement found on whether V2 is "GA/stable" vs "beta". The sst-to-anomalyco rename history and corporate details were not confirmed from a primary source.

## 2. Supported platforms and install methods; s390x and z/OS USS

### Takeaway
OpenCode ships prebuilt binaries only for macOS, Windows and Linux on x64 and ARM64 (glibc and musl). There is no s390x (Linux on IBM Z) build and no z/OS USS support. It is built on Bun, whose official platforms are also only x64 and ARM64. For a z/OS sysprog, the realistic setup is to run OpenCode on the workstation (macOS, Linux, or Windows/WSL) or an x86/ARM Linux jump host, and reach the mainframe via SSH, Zowe CLI or MCP.

### Cited Findings
- V1 install methods:
  - `curl -fsSL https://opencode.ai/install | bash`
  - `npm|bun|pnpm i -g opencode-ai`, `yarn global add opencode-ai`
  - `brew install anomalyco/tap/opencode` (recommended over the core formula)
  - Arch `pacman -S opencode` / `paru -S opencode-bin`
  - Windows `choco`, `scoop`, `mise use -g github:anomalyco/opencode`
  - `docker run -it --rm ghcr.io/anomalyco/opencode`
  - binaries from GitHub releases
  - Windows users are advised to use WSL

  — [OpenCode docs](https://opencode.ai/docs)
- V2 install methods:
  - `curl -fsSL https://opencode.ai/v2/install | bash`
  - `npm install -g @opencode/cli`
  - `brew install anomalyco/tap/opencode-v2`
  - `paru -S opencode-beta`, `mise use -g opencode`
  - Docker `ghcr.io/anomalyco/opencode:2.0.0`

  — [V2 docs](https://opencode.ai/v2/docs/). WinGet `winget install --id Anomaly.OpenCode -e` — [capitalandcompute.net](https://capitalandcompute.net/blog/opencode-2-0-what-changed/)
- V2 CLI binary targets: macOS (Apple silicon, Intel, Intel baseline); Windows (ARM64, x64, x64 baseline); Linux glibc and musl (ARM64, x64, x64 baseline). The desktop app ships .deb/.rpm/AppImage for x64 and ARM64. No s390x or ppc64le is listed — [V2 docs](https://opencode.ai/v2/docs/)
- V1 release assets seen include darwin-arm64/x64/x64-baseline and Linux desktop amd64 .deb and aarch64 .rpm. The full list of 40 assets was not visible — [GitHub releases](https://github.com/anomalyco/opencode/releases)
- Plugins are installed with Bun and the plugin context exposes "Bun's shell API". The runtime is Bun-based — [Plugins docs](https://opencode.ai/docs/plugins/)
- Bun's official platforms are Linux x64 (glibc, Nehalem or newer, needs SSE4.2) and ARM64, musl x64 and ARM64, Windows x64 and ARM64, and macOS 13+ (x64 and ARM64). s390x, ppc64le and big-endian are not mentioned. Recommended kernel is 5.6+, and it works down to 3.10 (RHEL 7) — [Bun installation docs](https://bun.sh/docs/installation)
- A search for OpenCode s390x found only x86_64 Arch packages and amd64/arm64 community Docker images. There is no s390x build or documented support — [Arch package](https://www.archlinux.org/packages/extra/x86_64/opencode), [Docker Hub mrlyc/opencode](https://hub.docker.com/r/mrlyc/opencode/tags)
- Related evidence on Bun portability: Bun binaries have crashed under qemu emulation on ppc64le (Claude Code SSH connector issue), and some x64 "baseline" builds have crashed with SIGILL on CPUs without AVX — [claudeissues.com ppc64le](https://claudeissues.com/issue/26332-feature-support-for-ppc64le-in-the-ssh-connector), [claudeissues.com AVX](https://claudeissues.com/issue/20116-bun-1-3-6-in-claude-code-2-1-15-crashes-with-sigill-on-cpus-without-avx-support)
- Recommended terminals are WezTerm, Alacritty, Ghostty and Kitty — [OpenCode docs](https://opencode.ai/docs)

### Inferences
- **z/OS USS: not feasible.** There is no Bun port for z/OS, and USS is EBCDIC and big-endian, so OpenCode cannot run natively there.
- **Linux on IBM Z (s390x): no official route.** Bun has no s390x target, so OpenCode would need an unofficial port or x86 emulation (qemu-user). Neither is supported, and the ppc64le evidence above suggests emulation is fragile.
- Recommended pattern: run OpenCode on a workstation or x86/ARM Linux host. Its `bash` tool can call Zowe CLI, `ssh` to USS, `ftp`/`sftp`, or `z/OSMF` REST via `curl`, gated by permission rules. Or wrap mainframe access as an MCP server (see section 5).
- Glibc 2.17 minimum and "baseline" x64 builds mean older RHEL 7/8 jump hosts are likely OK.

### Gaps
- No primary statement from OpenCode maintainers on s390x/ppc64le. No GitHub issues for s390x surfaced via web search; the GitHub search API was blocked in this session.
- Did not verify whether Bun has any experimental s390x work.

## 3. Model providers, local models and on-prem/air-gapped use

### Takeaway
OpenCode is provider-agnostic. It uses the Vercel AI SDK and Models.dev metadata to support "75+" providers, plus any OpenAI-compatible endpoint. Local models (Ollama, LM Studio, llama.cpp) are configured as custom OpenAI-compatible providers via `baseURL`, which is also how you point it at an on-prem gateway (vLLM, an internal LLM gateway, or presumably watsonx behind an OpenAI-compatible proxy). IBM watsonx is not a listed built-in provider.

### Cited Findings
- Uses the AI SDK and Models.dev for "75+ LLM providers" plus local models. Listed providers include Anthropic, OpenAI, Amazon Bedrock, Google Vertex AI, GitHub Copilot, Groq, DeepSeek and OpenRouter. **IBM watsonx is not listed** — [Providers docs](https://opencode.ai/docs/providers/)
- Local models (all via the `@ai-sdk/openai-compatible` npm package):

  | Server | baseURL |
  |---|---|
  | Ollama | `http://localhost:11434/v1` (raise `num_ctx` to 16k–32k if tool calls fail) |
  | LM Studio | `http://127.0.0.1:1234/v1` |
  | llama.cpp `llama-server` | `http://127.0.0.1:8080/v1` |

  Choose models with strong tool-calling support — [Providers docs](https://opencode.ai/docs/providers/)
- Custom OpenAI-compatible provider:
  1. `/connect` → Other → provider ID, then enter the key.
  2. In `opencode.json`, add `provider.<id>` with `npm: "@ai-sdk/openai-compatible"`, `name`, `options.baseURL`, and `models`.
  3. Optionally set `options.apiKey`, `options.headers`, and `limit.context` / `limit.output`.

  Use `@ai-sdk/openai` for endpoints that implement `/v1/responses` — [Providers docs](https://opencode.ai/docs/providers/)
- Credentials from `/connect` or `opencode auth login` are stored in `~/.local/share/opencode/auth.json` — [Providers docs](https://opencode.ai/docs/providers/), [CLI docs](https://opencode.ai/docs/cli/)
- Gateways are supported by overriding `baseURL` (Cloudflare AI Gateway, Vercel AI Gateway, Helicone, OpenRouter, LLM Gateway, Eden AI with an EU endpoint). `whitelist`/`blacklist` limit the `/models` picker — [Providers docs](https://opencode.ai/docs/providers/)
- Config `enabled_providers` (allowlist) and `disabled_providers` (blocklist, wins on conflict). `experimental.policies` can allow or deny `provider.use`. Managed config files (`/etc/opencode/` on Linux) enforce settings users cannot override — [Config docs](https://opencode.ai/docs/config/)
- Enterprise: central config can integrate with SSO and "ensures users reach only your internal AI gateway". Other providers can be disabled — [Enterprise docs](https://opencode.ai/docs/enterprise/)
- Air-gap relevant env vars:
  - `OPENCODE_DISABLE_MODELS_FETCH` (no fetch of models.dev)
  - `OPENCODE_MODELS_URL` (custom models catalog URL)
  - `OPENCODE_DISABLE_AUTOUPDATE`
  - `OPENCODE_DISABLE_LSP_DOWNLOAD`

  — [CLI docs](https://opencode.ai/docs/cli/)
- Corporate network: honours `HTTPS_PROXY`/`HTTP_PROXY`/`NO_PROXY` (set `NO_PROXY=localhost,127.0.0.1` so the TUI reaches its local server). Custom CA via `NODE_EXTRA_CA_CERTS` (PEM). For NTLM/Kerberos proxies, the docs suggest an LLM gateway — [Network docs](https://opencode.ai/docs/network/)
- V2 changes provider config to `providers` with `package: "aisdk:..."`, `settings.baseURL`, and `headers`/`body` — [V2 migration guide](https://opencode.ai/v2/docs/migrate-v1/)

### Inferences
- An air-gapped setup would combine:
  - a local or internal OpenAI-compatible endpoint (vLLM, Ollama, TGI, or a watsonx.ai deployment fronted by an OpenAI-compatible gateway such as LiteLLM)
  - `enabled_providers: ["<internal>"]`
  - `OPENCODE_DISABLE_MODELS_FETCH=1` and `OPENCODE_DISABLE_AUTOUPDATE=1`
  - `autoupdate: false`
  - `share: "disabled"`
  - `lsp` off or `OPENCODE_DISABLE_LSP_DOWNLOAD=1`
  - npm plugins pre-cached or avoided (they are installed via Bun from npm at startup)
- These settings can be enforced through `/etc/opencode/` managed config.
- Small local models often have weak tool calling, so expect a quality gap versus frontier models.

### Gaps
- No documentation found on a native watsonx provider. Whether the AI SDK community has a watsonx provider usable via `npm:` was not verified.
- No official "fully offline" guide found, and no full list of outbound endpoints contacted by default.

## 4. Configuration: opencode.json, AGENTS.md, agents, commands, permissions, tool restrictions

### Takeaway
V1 configuration is JSON/JSONC (`opencode.json`), merged from several layers from remote org config up to managed `/etc/opencode`. Project rules live in `AGENTS.md` (falling back to `CLAUDE.md`). Agents and commands are Markdown files with YAML frontmatter. Permissions are `allow`/`ask`/`deny` per tool, with glob patterns where the last match wins. Most permissions default to `allow`, so lock things down for mainframe work.

### Cited Findings
- **Config layers** (low to high precedence):
  1. remote `.well-known/opencode` org defaults
  2. `~/.config/opencode/opencode.json`
  3. `OPENCODE_CONFIG`
  4. project `opencode.json` (searched up to the git root)
  5. `.opencode/` dirs
  6. `OPENCODE_CONFIG_CONTENT`
  7. managed files (`/etc/opencode/` Linux, `/Library/Application Support/opencode/` macOS, `%ProgramData%\opencode` Windows)
  8. macOS MDM `ai.opencode.managed` (not user-overridable)

  Layers are merged, not replaced. `{env:VAR}` and `{file:path}` substitution is supported — [Config docs](https://opencode.ai/docs/config/)
- Other keys: `autoupdate` (true/false/"notify"), `small_model`, `instructions`, `tools`, `server`, `formatter`, `lsp`, `mcp`, `plugin`, `compaction`, `watcher.ignore`, and `snapshot` (default on; uses an internal git repo for rollback) — [Config docs](https://opencode.ai/docs/config/)
- **Rules:**
  - Project `AGENTS.md` (generated by `/init`; commit it) and global `~/.config/opencode/AGENTS.md`.
  - Claude Code fallbacks: `CLAUDE.md` and `~/.claude/CLAUDE.md`, plus skills from `~/.claude/skills/`. Disable with `OPENCODE_DISABLE_CLAUDE_CODE*`.
  - `instructions` array accepts paths, globs, or remote URLs (5 s timeout).

  — [Rules docs](https://opencode.ai/docs/rules/)
- **Agents:**
  - Primary agents: **Build** (default, all tools) and **Plan** (edits and bash default to `ask`; README describes plan as read-only and denying edits). Switch with Tab.
  - Subagents: **General**, **Explore** (read-only search), **Scout** (external docs research). Hidden agents: Compaction, Title, Summary.
  - Defined in JSON under `agent` or as Markdown in `.opencode/agents/` or `~/.config/opencode/agents/`.
  - Options: `description` (required), `model`, `temperature`, `top_p`, `prompt`, `steps`, `permission`, `mode` (primary/subagent/all), `hidden`, `disable`. `opencode agent create` generates one.

  — [Agents docs](https://opencode.ai/docs/agents/), [GitHub README](https://github.com/sst/opencode)
- **Custom commands:**
  - Markdown in `.opencode/commands/` or `~/.config/opencode/commands/` (filename = `/command`), or JSON under `command` with a `template`.
  - Frontmatter: `description`, `agent`, `model`, `subtask`.
  - Placeholders: `$ARGUMENTS`, `$1..$n`, `` !`shell cmd` `` (injects output), `@file`.
  - A custom command overrides a built-in of the same name.

  — [Commands docs](https://opencode.ai/docs/commands/)
- **Permissions (V1):**
  - Key is `permission`; the old boolean `tools` is deprecated.
  - Keys: `read`, `edit` (covers edit/write/patch), `glob`, `grep`, `bash` (matches parsed commands), `task`, `skill`, `lsp`, `question`, `webfetch`, `websearch`, `external_directory`, `doom_loop`.
  - Patterns use `*` and `?`, and `~`/`$HOME` expand. **Last matching rule wins**, so put `"*"` first.
  - Defaults: mostly `allow`. `doom_loop` and `external_directory` default to `ask`. `*.env` reads are denied.
  - "Ask" responses: once, always (this session), or reject.
  - `--auto` approves all asks but still enforces `deny`. Per-agent overrides go in `agent.<name>.permission`, and agent rules take precedence. `OPENCODE_PERMISSION` env injects inline JSON.

  — [Permissions docs](https://opencode.ai/docs/permissions/), [CLI docs](https://opencode.ai/docs/cli/)
- Example from docs: `"bash": {"*":"ask","git *":"allow","git push *":"deny"}`, `"edit": {"*":"deny", "<path>":"allow"}` — [Permissions docs](https://opencode.ai/docs/permissions/)
- **V2 changes:**
  - An ordered `permissions` array of `{action, resource, effect}`. Actions are renamed: `bash` → `shell`, `task` → `subagent`, write/patch → `edit`.
  - **`external_directory` is allowed by default in V2** (it was ask in V1).
  - `agent`/`mode` → `agents`, `command` → `commands`, `autoshare` → `share`. `tui.json` → `cli.json`.

  — [V2 migration guide](https://opencode.ai/v2/docs/migrate-v1/), [capitalandcompute.net](https://capitalandcompute.net/blog/opencode-2-0-what-changed/)

### Inferences
- For mainframe work, a sensible policy:
  - `"permission": {"*":"ask"}` baseline
  - `bash` allowlisting read-only Zowe commands (e.g. `zowe zos-files list *`, `zowe zos-jobs view *`)
  - `ask` for job submission and dataset writes (`zowe zos-jobs submit *`, `zowe zos-files upload *`)
  - `deny` for destructive commands (`zowe zos-files delete *`, `tso *` operator commands)
  - `webfetch` set to ask or deny
- Ship this policy via `/etc/opencode/` managed config so users can't loosen it.
- Pattern matching on parsed commands is not a security sandbox (shell metacharacters, scripts or aliases could bypass it). Pair it with RACF least-privilege on the TSO/z/OSMF user that the agent's credentials use.
- `AGENTS.md` is the natural place for site conventions: dataset naming (HLQs), JCL standards, "never submit to production LPAR", SMP/E practices.
- Anyone upgrading to V2 must rewrite permission blocks. `external_directory` silently becomes allow.

### Gaps
- Whether `bash` patterns match commands inside pipelines and subshells was not detailed in the docs fetched.

## 5. Extensibility: MCP, plugins, LSP, custom tools

### Takeaway
V1 supports local (stdio) and remote (HTTP, with automatic OAuth/DCR) MCP servers, JS/TS plugins with lifecycle hooks and custom tools, and 34 built-in LSP servers plus custom ones. LSP is off by default. V2 breaks V1 plugins and currently does not run LSP at all.

### Cited Findings
- **MCP config** goes under `mcp.<name>`:
  - Local: `type:"local"`, `command` (array), `cwd`, `environment`, `enabled`, `timeout` (default 5000 ms).
  - Remote: `type:"remote"`, `url`, `headers`, `oauth` (object, or `false`), `enabled`, `timeout`.
- **MCP auth:** OAuth starts automatically on a 401, with Dynamic Client Registration (RFC 7591). Pre-registered `clientId`/`clientSecret`/`scope` are also supported. Tokens are stored in `~/.local/share/opencode/mcp-auth.json`. CLI: `opencode mcp add|list|auth|logout|debug`.
- **MCP tool control:** Tools are prefixed `servername_*` and can be disabled by glob via `tools`, then re-enabled per agent. The docs warn that MCP tools consume context (the GitHub MCP server can exceed limits).

  — [MCP docs](https://opencode.ai/docs/mcp-servers/), [CLI docs](https://opencode.ai/docs/cli/)
- **Plugins:**
  - Loaded from `.opencode/plugins/`, `~/.config/opencode/plugins/`, or the npm packages in the `plugin` array. npm plugins are installed with Bun and cached in `~/.cache/opencode/node_modules/`.
  - Context: `project`, `directory`, `worktree`, `client` (SDK), `$` (Bun shell).
  - Events: `session.idle/created/error`, `file.edited`, `message.updated`, `permission.asked`, `shell.env`, `tool.execute.before/after`, `experimental.session.compacting`.
  - Custom tools use the `tool()` helper from `@opencode-ai/plugin`. A plugin tool overrides a built-in tool of the same name. The docs include an example that blocks `.env` reads via `tool.execute.before`.
  - `--pure` runs with no external plugins, and `OPENCODE_DISABLE_DEFAULT_PLUGINS` exists.

  — [Plugins docs](https://opencode.ai/docs/plugins/), [CLI docs](https://opencode.ai/docs/cli/)
- **LSP:** 34 built-ins (clangd, jdtls, pyright, bash, yaml-ls, typescript, gopls, etc.). Eleven auto-download (some from GitHub releases), which `OPENCODE_DISABLE_LSP_DOWNLOAD=true` turns off. **LSP is disabled by default.** Enable with `"lsp": true`. Custom servers take `command` + `extensions` (+ `env`, `initialization`) — [LSP docs](https://opencode.ai/docs/lsp/)
- **V2:**
  - MCP moves under `mcp.servers`. `enabled:false` → `disabled:true` (a naive find-replace inverts meaning). Timeout splits into `timeout.catalog`/`timeout.execution`, and OAuth fields become snake_case.
  - "V1 plugin implementations do not run in V2". Plugins must be ported.
  - V2 keeps `lsp` config but **runs no language servers**.

  — [V2 migration guide](https://opencode.ai/v2/docs/migrate-v1/)
- Third-party ecosystem example: oh-my-opencode/oh-my-openagent plugins use PostHog analytics (opt-out via `OMO_DISABLE_POSTHOG` or `"telemetry": false`) — [oh-my-openagent privacy policy](https://raw.githubusercontent.com/code-yeongyu/oh-my-openagent/dev/docs/legal/privacy-policy.md)

### Inferences
- **MCP:** Zowe-based or z/OSMF-based MCP servers (IBM and community) are the cleanest way to give OpenCode structured, permission-scoped access to datasets, jobs and consoles, rather than raw `bash`. Configure them as `type:"local"` with credentials in `environment` via `{env:...}`.
- **LSP:** IBM's COBOL/PL/I/HLASM language servers (from Z Open Editor) are not in the built-in list. Wiring them in as custom LSP servers (`extensions: [".cbl",".cob",".cpy",".pli",".asm",".jcl"]`) is plausible but unverified (these servers may be packaged only as VS Code extensions). This only works in V1.
- **Plugins:** Plugins can enforce site guardrails, e.g. a `tool.execute.before` hook that blocks `zowe ... --host PROD*`.
- **Third-party plugins:** These may add their own telemetry, so vet them.

### Gaps
- The standalone "Custom Tools" doc page (`.opencode/tools/`) was not fetched.
- No direct evidence found of anyone using OpenCode with Zowe, z/OSMF, or IBM Z language servers. A search linking OpenCode to IBM Z/COBOL returned only IBM watsonx Code Assistant for Z material — [IBM Community blog](https://community.ibm.com/community/user/blogs/latrell-freeman/2025/12/10/reimagining-mainframe-development-with-agentic-ai)

## 6. Non-interactive and scripting use (run, server, SDK), GitHub integration

### Takeaway
`opencode run` gives one-shot or scripted use with JSON output. `opencode serve` exposes an HTTP API (OpenAPI 3.1 at `/doc`, default 127.0.0.1:4096) that the TUI itself uses. `opencode acp` speaks the Agent Client Protocol. A GitHub Action / GitHub App responds to `/opencode` or `/oc` comments. V2 replaced the server API and client package.

### Cited Findings
- `opencode run [message]` flags: `--command`, `-c/--continue`, `-s/--session`, `--fork`, `-m/--model`, `--agent`, `-f/--file`, `--format json`, `--title`, `--attach <url>` (to a running server), `-u/-p` basic auth, `--dir`, `--port`, `--variant`, `--thinking`, `--auto` — [CLI docs](https://opencode.ai/docs/cli/)
- Other commands:
  - Servers: `serve` (headless API), `web` (API + web UI), `acp` (Agent Client Protocol over stdio, nd-JSON), `attach [url]`.
  - Sessions and data: `session list|delete`, `export [id] --sanitize` / `import`, `stats`, `db [query]`.
  - Setup and lifecycle: `models --refresh`, `agent create`, `plugin <module>`, `pr <number>`, `upgrade`, `uninstall`.

  — [CLI docs](https://opencode.ai/docs/cli/)
- **Server API:**
  - Default `127.0.0.1:4096`, OpenAPI 3.1 spec at `/doc`. Basic auth via `OPENCODE_SERVER_PASSWORD` (user `opencode`, override with `OPENCODE_SERVER_USERNAME`).
  - Endpoints: `/global/health`, `/session`, `/session/:id/message`, `/session/:id/prompt_async`, `/session/:id/abort`, `/event` (SSE), `/config`, `/provider`, `/file/content`, `/find`, `/tui/submit-prompt`.
  - Running `opencode` starts a TUI plus a server, and the TUI is a client of that server. IDE plugins use the `/tui` endpoints.
  - The SDK is generated from the OpenAPI spec; a separate `/docs/sdk` page exists.

  — [Server docs](https://opencode.ai/docs/server/)
- Server exposure risk: `server.mdns` makes the server discoverable on the LAN. With mDNS on and no hostname set, it binds `0.0.0.0` — [Config docs](https://opencode.ai/docs/config/)
- V2: the server API was revised (JSON over HTTP + SSE), and clients use the `@opencode/client` package. V1 `server` config is ignored with a warning — [V2 migration guide](https://opencode.ai/v2/docs/migrate-v1/), [capitalandcompute.net](https://capitalandcompute.net/blog/opencode-2-0-what-changed/)
- **GitHub integration:**
  - Setup: `opencode github install` (installs the GitHub app github.com/apps/opencode-agent, adds a workflow, and sets secrets), or a manual `.github/workflows/opencode.yml` using `anomalyco/opencode/github@latest` with `model: provider/model` and the API key in env.
  - Triggers: `/opencode` or `/oc` in issue, PR or line comments (configurable via `mentions`).
  - Events: `issue_comment`, `pull_request_review_comment`, `issues`, `pull_request`, `schedule`, `workflow_dispatch`.
  - Permissions: `id-token: write` (OIDC) unless `use_github_token: true`, plus `contents`/`pull-requests`/`issues: write`.
  - A GitLab page also exists.

  — [GitHub docs](https://opencode.ai/docs/github/)

### Inferences
- `opencode run --format json --agent <readonly-agent> -m <internal model>` can be scripted for batch tasks: summarising SYSLOG/JES output, explaining JCL or COBOL, drafting SMP/E HOLDDATA analyses. It can be scheduled from cron on a Linux host or a CI pipeline (e.g. Jenkins/GitLab for z/OS DevOps via DBB).
- `--auto` with `opencode run` approves all "ask" prompts. For unattended runs, rely on explicit `deny` rules.
- The SDK package name `@opencode-ai/sdk` (V1) was not confirmed from the server page itself. V2 uses `@opencode/client`.

### Gaps
- The SDK page and GitLab page were not fetched. Whether the GitHub Action works against GitHub Enterprise Server (common at banks) is unverified.

## 7. Privacy and data handling: what is sent where, telemetry, sharing, and how to disable

### Takeaway
OpenCode states it does not store code or context. Prompts and code go directly from your machine to the configured model provider (or your gateway). The main egress risks are:
- the share feature (V1 uploads full conversation history to a public link; disable with `"share": "disabled"`)
- the auto-update check
- the models.dev catalog fetch
- LSP auto-downloads
- npm plugin installs
- remote instruction URLs or MCP servers
- the hosted Zen/Console services, if you choose them

No core telemetry switch is documented. The privacy policy covers hosted services and the website (Google Analytics cookies), not the self-run open-source CLI.

### Cited Findings
- "OpenCode does not store your code or context data." Processing is local or via direct API calls to your provider or internal gateway. The Enterprise page recommends "disabling the share pages to ensure your data never leaves your organization". Self-hosted share pages are on the roadmap, not available — [Enterprise docs](https://opencode.ai/docs/enterprise/)
- **V1 share modes:** `"share": "manual"` (default, via `/share`), `"auto"`, or `"disabled"`. Shared links are public to anyone with the URL and include full conversation history, file diffs and tool-use records. `/unshare` removes the link and deletes the data — [Mintlify mirror of OpenCode share docs](https://www.mintlify.com/anomalyco/opencode/share), [learnopencode.com](https://learnopencode.com/en/5-advanced/16-share) (secondary sources; the official opencode.ai/docs/share URL returned 404 at fetch time)
- **V2 share status:** Sharing is not yet available in V2, and the `share` field is parsed but not acted on. The V2 docs advise against treating it as a privacy control — per search summary of [V2 sharing docs](https://v2.opencode.ai/docs/sharing) (direct fetch of opencode.ai/v2/docs/sharing/ returned 404; treat as moderately confident)
- **Privacy policy** (effective 6 Mar 2026): self-run open-source software is governed by the repo license, not the policy. For hosted services, prompts and conversations are "Passing through to upstream provider... Not stored". The website uses analytics cookies (e.g. Google Analytics). Downloaded apps may transmit device info such as update availability. IP and device ID may be used for internal analytics — [Privacy policy](https://opencode.ai/legal/privacy-policy)
- **Documented disable switches:**
  - `OPENCODE_DISABLE_AUTOUPDATE` and config `autoupdate: false`
  - `OPENCODE_DISABLE_MODELS_FETCH` / `OPENCODE_MODELS_URL`
  - `OPENCODE_DISABLE_LSP_DOWNLOAD`
  - `OPENCODE_DISABLE_DEFAULT_PLUGINS`, `--pure`
  - `OPENCODE_DISABLE_CLAUDE_CODE*`
  - Exa/Parallel web search are opt-in (`OPENCODE_ENABLE_EXA`/`OPENCODE_ENABLE_PARALLEL`)

  — [CLI docs](https://opencode.ai/docs/cli/)
- No telemetry environment variable or config option is documented in the CLI or config docs. V2 lists an `openTelemetry` V1 field as ignored, which implies V1 had an OpenTelemetry option (likely for the user's own tracing export) — [CLI docs](https://opencode.ai/docs/cli/), [V2 migration guide](https://opencode.ai/v2/docs/migrate-v1/)
- **Local data stores:**
  - V1 credentials: `~/.local/share/opencode/auth.json` and `mcp-auth.json`, stored in plain files.
  - Snapshots: an internal git repo, for rollback.
  - V2: sessions in `~/.local/share/opencode/opencode.db`.
  - `.env` files are denied from `read` by default.

  — [Providers docs](https://opencode.ai/docs/providers/), [MCP docs](https://opencode.ai/docs/mcp-servers/), [Config docs](https://opencode.ai/docs/config/), [Permissions docs](https://opencode.ai/docs/permissions/), [capitalandcompute.net](https://capitalandcompute.net/blog/opencode-2-0-what-changed/)
- The web search tool and `webfetch` can send URLs and queries outward; both are permissionable — [Permissions docs](https://opencode.ai/docs/permissions/)

### Inferences
- **What goes to the model:** Anything the agent reads (source, JCL, SYSOUT, dataset contents with possible PII/PCI data, RACF output) is sent to the model provider. For regulated sites, the model endpoint must be internal or contractually approved.
- **Hardening checklist for a z/OS shop**, enforced via `/etc/opencode/opencode.json` managed config:
  - `share: "disabled"`
  - `autoupdate: false`
  - `enabled_providers` limited to the internal gateway
  - `permission.webfetch`/`websearch: deny`
  - env `OPENCODE_DISABLE_MODELS_FETCH=1`, `OPENCODE_DISABLE_LSP_DOWNLOAD=1`, `OPENCODE_DISABLE_CLAUDE_CODE=1`
  - no npm plugins (or a vetted internal registry)
  - `HTTPS_PROXY` + `NODE_EXTRA_CA_CERTS`
  - server bound to 127.0.0.1 with mDNS off
- Also protect `auth.json` (file permissions) and keep mainframe credentials out of prompts.

### Gaps
- No authoritative statement found on whether the V1 or V2 CLI sends any usage telemetry to Anomaly by default. Verify by network capture or a source review before approving for regulated use.
- No full list of default outbound hosts (likely opencode.ai for update checks, models.dev, GitHub for LSP and releases, npm registry for plugins). This is inferred, not documented.
- Official V1 share doc page could not be fetched directly (404). Share retention period is undocumented.
