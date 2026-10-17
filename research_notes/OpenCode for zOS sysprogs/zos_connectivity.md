# Connecting a terminal AI coding agent (OpenCode) to IBM z/OS: technical bridges (as of Oct 2026)

Legend: **[OFFICIAL]** means a vendor or Open Mainframe Project/Zowe project. **[COMMUNITY]** means an individual or third-party project. A claim marked "(prior knowledge, not re-fetched)" comes from well-established documentation that was not re-verified in this session. The report writer should treat those claims as lower-confidence.

## Q1. Zowe CLI: install, team config, key commands, JSON output, mainframe requirements

### Takeaway
Zowe CLI (v3.x LTS, current docs stream v3.6.x) is the most mature agent-callable bridge. OpenCode can run it through its shell tool. It has a global `--response-format-json`/`--rfj` flag for machine-readable output. Most command groups need z/OSMF on the mainframe. `zos-ssh` needs only an SSH login, and `auth` uses the API Mediation Layer. A z/OSMF-free path over SSH (Zowe Remote SSH, "zowex") is maturing; 3.6 was targeted for GA.

### Cited Findings
- Docs show **v3.6.x LTS** as current, with v3.5.x, 3.4.x, 3.3.x, 3.2.x and v2.18.x LTS also listed. "Support for Zowe Version 2 ends March 30, 2027", and V3 enters maintenance March 1, 2027 — [Zowe docs: team profiles](https://docs.zowe.org/stable/user-guide/cli-using-using-team-profiles)
- Team config concepts:
  - **Config files:** team config (shared, most common), user config (rarely needed), global config (in `ZOWE_CLI_HOME`, default `~/.zowe`) and project config (a chosen directory; Zowe Explorer also uses it when that directory is a VS Code workspace).
  - **Profile types:** service profiles (e.g. z/OSMF; plug-ins add others such as `cics`), base profiles (`global_base`/`project_base`, which can hold API ML tokens for MFA/SSO) and parent profiles.
  - **Secrets:** `zowe config secure [--global-config]` stores them in the secure credential store.
  — [Zowe docs: team profiles](https://docs.zowe.org/stable/user-guide/cli-using-using-team-profiles)
- Core command groups:
  - `zos-files`: create, download, upload and list data sets, plus VSAM.
  - `zos-jobs`: submit JCL; list jobs, spool files and status.
  - `zos-tso`: issue TSO commands, run REXX, manage TSO address spaces.
  - `zos-console`: console commands through an extended MCS console. The docs say security admins should control which commands are allowed.
  - `zos-ssh`: USS shell commands over SSH. It was "previously named zos-uss" and needs the user ID to be allowed SSH login.
  - Others: `zos-logs` (operlog over a time frame), `zos-workflows` (needs z/OSMF), `provisioning` (z/OSMF Software Services), `zosmf` (z/OSMF info, profile check), `auth` (API ML token login/logout), `config`, `plugins`, `daemon` (background processor for speed).
  — [Zowe docs: core command groups](https://docs.zowe.org/stable/user-guide/cli-using-understanding-core-command-groups)
- The global JSON flag is defined in Zowe Imperative as `JSON_OPTION = "response-format-json"`, alias `rfj` — [zowe-cli source, Constants.ts](https://raw.githubusercontent.com/zowe/zowe-cli/master/packages/imperative/src/constants/src/Constants.ts)
- Download options in `zos-files download` include `--binary/-b` (no conversion), `--record/-r` (no conversion, record length prepended), `--encoding/-ec` (convert using a given codepage), `--file/-f`, `--extension/-e`, `--directory/-d`, `--volume-serial/-vs`, `--extension-map`, `--max-concurrent-requests`, `--preserve-original-letter-case`, `--fail-fast`, `--attributes`, `--overwrite` and `--include-hidden` — [Download.options.ts](https://raw.githubusercontent.com/zowe/zowe-cli/master/packages/cli/src/zosfiles/download/Download.options.ts), [en.ts strings](https://raw.githubusercontent.com/zowe/zowe-cli/master/packages/cli/src/zosfiles/-strings-/en.ts)
- New in Zowe 3.5 (blog last updated 2026-06-09):
  - A `certFile` profile property references certificates in the Windows/macOS certificate store by common name.
  - USS upload gains `--make-dirs` and `--skip-directory-check`.
  - `zos-ssh` gains `--exec`, which skips shell init and profile loading for faster runs.
  - Team Config properties can reference environment variables, and secure values are never logged or displayed.
  — [OMP: What's new in Zowe 3.5](https://openmainframeproject.org/blog/whats-new-in-zowe-3-5/)
- Zowe 3.5 also brings Zowe Remote SSH (zowex), which accesses jobs and data sets over SSH instead of z/OSMF. The 3.6 goal was GA and integration into Zowe Explorer, so users can choose z/OSMF or SSH — [OMP: Zowe 3.5](https://openmainframeproject.org/blog/whats-new-in-zowe-3-5/)
- Typical install and usage, (prior knowledge, not re-fetched; see [docs.zowe.org CLI install](https://docs.zowe.org/stable/user-guide/cli-installcli)):
  - Install with `npm install -g @zowe/cli@zowe-v3-lts`.
  - Create a config with `zowe config init` (writes `zowe.config.json`; add `--global-config` for the global file).
  - Per-command overrides come from `ZOWE_OPT_*` environment variables, e.g. `ZOWE_OPT_HOST`, `ZOWE_OPT_USER`, `ZOWE_OPT_PASSWORD`.
  - Example commands: `zowe files list ds "HLQ.*" --rfj`, `zowe jobs submit ds "HLQ.JCL(X)" --wait-for-output --rfj`, `zowe jobs view sfbi <jobid> <id>`, `zowe tso issue command "LISTDS 'X'" --ssm`, `zowe console issue command "D A,L"`.
- Mainframe-side z/OS requirements for Zowe server components:
  - z/OS in active support, e.g. 2.5 or 3.1; Zowe 2.11+ is needed on z/OS 3.1.
  - Node.js 18/20/22 for the app-server.
  - IBM Semeru Java 17.
  - About 1200 MB zFS.
  - OpenSSH is "optional but recommended".
  — [Zowe docs: z/OS system requirements](https://docs.zowe.org/stable/user-guide/systemrequirements-zos)
- Zowe CLI itself does **not** require the Zowe server (API ML/ZSS) on z/OS. Its core z/OS groups talk to z/OSMF REST directly, and API ML is optional for SSO/MFA tokens — (inferred from the `auth` group needing API ML and the z/OSMF requirements in [core command groups](https://docs.zowe.org/stable/user-guide/cli-using-understanding-core-command-groups))

### Inferences
- For an agent, Zowe CLI with `--rfj` is the lowest-friction path. It is just a shell tool call, the output parses as JSON, credentials stay in the OS keyring via team config, and site-specific plug-ins (CICS, Db2, Endevor, etc.) extend the surface.
- Zowe CLI startup latency (Node) adds up over many agent calls. `zowe daemon` mode mitigates it.
- Agents should be told to always pass `--rfj` and to use `--binary`/`--record` or explicit `--encoding` when round-tripping data that is not plain text.

### Gaps
- The official CLI command reference (web_help) did not render through the fetch tool, so the full option list for each command (e.g. `jobs submit --wait-for-output`) was not re-verified this session.
- I did not confirm whether Zowe 3.6 shipped Remote SSH GA as planned.

## Q2. Zowe Explorer, Client SDKs, Zowe Remote SSH (zowex): can an agent use them?

### Takeaway
Zowe Explorer is a VS Code/IntelliJ GUI and cannot be driven directly from OpenCode. Its value to an agent is the shared team config, and it can host MCP servers (see Q3). The Zowe Node SDKs (`@zowe/zos-files-for-zowe-sdk`, etc.) and the Python and Kotlin SDKs can be scripted by an agent. Zowe Remote SSH (repo `zowe/zowe-native-proto`, CLI `zowe zssh`, binary `zo`) is the newer SSH-native path and the foundation of the Zowe MCP server.

### Cited Findings
- Zowe Remote SSH is "an open-source, native protocol for z/OS mainframe operations via SSH with minimal server-side configuration".
  - **Architecture:** a C++/Metal C z/OS server, with a CLI binary `zo`, an I/O server `zo server` and an APF-authorized `zoa` binary for MVS console commands.
  - **Clients:** a Node.js SDK, a Zowe CLI plug-in (`zowe zssh`) and a Zowe Explorer extension. A Python REST API is experimental.
  - **Output:** every command accepts `--json`.
  - **Requirements:** z/OS 2.5+, current LE maintenance (APARs listed), an SSH login and a writable USS directory. Building needs IBM Open XL C/C++ 2.1 and `xlc` for Metal C. Prebuilt binaries come via `npm run download`.
  - **Status:** no tagged releases visible, 162 open issues, EPL-2.0.
  — [github.com/zowe/zowe-native-proto](https://github.com/zowe/zowe-native-proto)
- Zowe Explorer 3.5 (VS Code) adds multi-member search, download from the MVS/USS tree, job submit with completion polling, inline PDS member filter and VSAM favorites. The IntelliJ plug-in and its Kotlin SDK are being refreshed for 3.6 — [OMP: Zowe 3.5](https://openmainframeproject.org/blog/whats-new-in-zowe-3-5/)
- Zowe Explorer reads project team config when the folder is opened as a VS Code workspace. The same `zowe.config.json` therefore serves both the human's Explorer and the agent's CLI — [Zowe docs: team profiles](https://docs.zowe.org/stable/user-guide/cli-using-using-team-profiles)

### Inferences
- A shared project `zowe.config.json` in the repo, with secrets in the keyring, lets a sysprog and OpenCode work against the same profiles.
- Sites without z/OSMF (or that dislike its footprint) can use zowex over SSH, but it is still pre-GA software built from source.

### Gaps
- I did not verify the published npm names, versions or maturity of the Zowe Python SDK and the Node SDK packages in 2026.

## Q3. MCP servers for z/OS and mainframes

### Takeaway
As of 2026 there are several real options:
- **Zowe MCP Server** (official OMP/Zowe, early-stage, SSH via zowex, stdio).
- **IBM Z Open Editor / IBM Developer for z/OS on VS Code MCP server** (official IBM, about 62 tools, local streamable-HTTP, needs the VS Code host running).
- **IBM RSE API 1.2.5 built-in MCP endpoint** (official IBM, server-side on z/OS, licensed).
- **IBM z/OS Connect MCP server** (official IBM, turns deployed z/OS Connect APIs into MCP tools).
- **Broadcom Code4z Assistant** (Endevor-aware MCP, beta Jan 2026 per a LinkedIn self-description).
- Assorted community servers on z/OSMF, RSE or FTP.

OpenCode can use any of them. stdio servers use `type: "local"`; HTTP servers use `type: "remote"` with headers.

### Cited Findings
**Zowe MCP Server [OFFICIAL – Zowe/OMP; contributors from Broadcom and IBM]** — [github.com/zowe/zowe-mcp](https://github.com/zowe/zowe-mcp)
- MCP server "for z/OS data sets, jobs, and UNIX System Services". It works with Claude Code, Cursor, VS Code, Zed and Roo Code, and has an optional VS Code extension for Copilot Chat.
- **Tools:**
  - Context and system: `getContext`, `setSystem`, list systems.
  - Data sets: list, read, write, create, delete, copy, rename, restore, search, attributes (e.g. `listDatasets`).
  - USS: list, read, write, create, delete, chmod, chown, chtag, copy.
  - TSO and console commands.
  - Jobs: submit, status, list, output, cancel, hold, release, delete.
  - Editor-open helpers.
  - CLI `call-tool` and `init-mock`.
- **Transport:** SSH via the Zowe Remote SSH SDK (`zowex-sdk`). Run as `node dist/index.js --stdio --native --system USER@host[:port]`, with multiple systems via `--config` JSON. `ZOWE_MCP_REQUIRE_EXPLICIT_SYSTEM=1` forces an explicit target.
- **Auth order:** SSH key, then a password env var (`ZOWE_MCP_PASSWORD_<USER>_<HOST>` or the `ZOWE_MCP_CREDENTIALS` map), then Vault KV, then an interactive prompt. ssh-agent is not supported yet.
- **Capability tiers:** `read-strict` (default), `read`, `update`, `delete` and `full` (job submit and commands). TSO and USS commands also pass pattern-based block/elicit/allow gates. Tool output is marked as untrusted data against prompt injection (`ZOWE_MCP_DATA_MARKING`, default on).
- **Mock mode** (`init-mock`, `--mock <dir>`) is for CI with no mainframe.
- **Packaging:** needs Node 22+. `@zowe/mcp-server` is **not yet on public npm**, so install from a tarball (`npm run pack:server`) or a CI artifact.
- **Maturity:** about 281 commits, 20 stars, no releases visible, EPL-2.0.
- [PR #126](https://github.com/zowe/zowe-mcp/pull/126) proposes running the MCP server **on z/OS itself**, with a SAF-backed OAuth IdP, same-system zowex and AT-TLS enforcement. I did not confirm whether it has merged.
- OMP's write-up of Joe Winchester's (IBM STSM) GS UK 2026 session (Apr 21–23, 2026) describes these risk tiers: read-only runs unprompted, side effects (job submit) need confirmation and destructive operations get extra checks. It calls the project "early stage" and is soliciting sysprog input on guardrails. Its extensibility is designed "with the Zowe CLI plugin ecosystem in mind" — [OMP blog](https://openmainframeproject.org/blog/using-zowe-in-an-ai-world-with-mcp-agents/)
- Zowe 3.5 notes describe zowex as "the foundation for the Zowe MCP server" — [OMP: Zowe 3.5](https://openmainframeproject.org/blog/whats-new-in-zowe-3-5/)
- The project was featured at the OMP IBM TechXchange 2026 pre-conference — [OMP TechXchange 2026](https://openmainframeproject.org/announcements/open-mainframe-project-at-ibm-techxchange-2026/)

**IBM Developer for z/OS on VS Code / IBM Z Open Editor MCP server ("agent mode") [OFFICIAL – IBM]**
- Exposes data set read/update, job submit, job info and spool, z/OS UNIX home files, plus DBB build, Wazi Deploy, ZCodeScan, RSE and ZAPP tools.
- **Example tool names:** `zowe-get-members`, `zowe-download-member`, `zowe-submit-job-dataset`, `zowe-submit-job-jcl`, `zowe-list-all-profiles`, `zopeneditor-get-default-zowe-profile`, `zapp-get-property-groups`, `zopeneditor-read-the-log`, `wazideploy-check-version`, `dbb-get-build-result`.
- **Setup:** enable `zopeneditor.mcp.enabled` (off by default). Add `.vscode/mcp.json` or `.bob/mcp.json` with a server name containing "zopeneditor" and a unique localhost port per workspace (e.g. `http://localhost:3005/mcp`). IBM advises against auto-approving write tools.
- **Requirements:** a default Zowe profile, plus remote DBB, Db2 config and an RSE profile for build tools.
- **Clients:** it supports IBM Bob, Bob Shell, Copilot and "any MCP client that supports the http-streaming protocol".
— [IBM Docs: agent mode](https://www.ibm.com/docs/en/developer-for-zos/17.0.x?topic=overview-agent-mode); listed in [github.com/IBM/mcp](https://github.com/IBM/mcp)
- An IBM Developer article (16 May 2026) shows the `zopeneditor-local` server at `localhost:63299/mcp` reporting "Running | 62 tools". Named tools include `userbuild-run-user-build`, `zowe-submit-dataset` and `zowe-get-all-job-spools-output` — [IBM Developer: Automate IBM Z workflows with Bob and MCP](https://developer.ibm.com/articles/mcp-bob-ibm-z-workflow-execution/)

**IBM RSE API 1.2.5 built-in MCP endpoint [OFFICIAL – IBM]**
- Endpoint `/rseapi/mcp` on the existing RSEAPI port (examples 6800/6804/6805), using streamable HTTP.
- Tool categories: UNIX operations, JES operations, MVS data set operations and administration. Tools are prefixed by hostname when there are multiple servers.
- Auth uses an `Authorization` header (Basic or Bearer).
- Enable with `RSEAPI_MCP_ENABLE=true` in `rseapi.env`, then restart; verify with `/serverinfo`.
- Requires an **IDzEE or ADFz VU license**. Post dated June 2026.
— [IBM Community: RSE API 1.2.5](https://community.ibm.com/community/user/blogs/dave-mcknight1/2026/06/19/rse-api-125)

**IBM z/OS Connect MCP server [OFFICIAL – IBM]**
- Runs inside z/OS Connect 3.0 over Streamable HTTP. Each operation of a deployed API is registered automatically as an MCP tool, and tools update on deploy/undeploy.
- The intro page does not cover auth; there is a separate "Configuring MCP for IBM z/OS Connect" page.
— [IBM Docs: z/OS Connect MCP](https://www.ibm.com/docs/en/zos-connect/3.0.0?topic=connect-what-is-mcp)
- This is the route to CICS, IMS and Db2 *business* APIs, not to sysprog operations.

**IBM MCP catalog:** [github.com/IBM/mcp](https://github.com/IBM/mcp) lists only the IDz-on-VS-Code and z/OS Connect servers for Z. It has no Db2-for-z/OS, CICS or watsonx Code Assistant for Z MCP entries.

**Broadcom [OFFICIAL-ish, unconfirmed]:** a Broadcom architect's LinkedIn profile mentions a "Code4z Assistant" MCP server making AI assistants "Endevor-aware", with a beta in January 2026 — [LinkedIn: Petr Plavjaník](https://cz.linkedin.com/in/plavjanik). I found no Broadcom press release confirming its status.

**Community servers [COMMUNITY – unvetted]:**
- `NotHarshhaa/mainframe-mcp-server`, npm `@notharshhaa/mainframe-mcp-server`, listed as v2.0–2.5. Covers jobs, data sets, USS and diagnostics via the Zowe SDK and z/OSMF (`ZOSMF_HOST`, user/password or token). Advertises read-only mode, tool allow/block lists and audit logging — [claudemarket listing](https://claudemarket.ai/mcp/NotHarshhaa/mainframe-mcp-server), [mcpmarket](https://mcpmarket.com/server/mainframe)
- "Zowe CLI MCP" wrapper — [mcp.so](https://mcp.so/servers/zowe-cli-mcp)
- An RSE-API-based "Mainframe MCP Server" (rohittrimale), a z/OS **FTP** MCP server (arunkumars-mf; read-only by default) and others — [glama listings](https://glama.ai/mcp/servers/e2odem095k)
- An Endevor MCP with 43 tools on Endevor REST API v2. Ownership is unclear, and it is bundled with a pentest tool (a red flag) — [glama](https://glama.ai/mcp/servers/yzc1b7oe5f)
- `ExpertVagabond/ibmz-mcp-server`: Key Protect HSM plus z/OS Connect calls, 12 tools, "unvalidated" — [GitHub](https://github.com/ExpertVagabond/ibmz-mcp-server)

**OpenCode config:**
- MCP servers go under `mcp` in `opencode.json`:
  - `type: "local"` takes `command` (array), `environment`, `cwd`, `timeout` (default 5000 ms) and `enabled`.
  - `type: "remote"` takes `url`, `headers` (e.g. `"Authorization": "Basic {env:RSE_AUTH}"`), `oauth` and `enabled`.
- Tools are prefixed by server name and can be disabled globally with globs (`"tools": {"zowe*": false}`), then re-enabled per agent.
— [OpenCode docs: MCP servers](https://opencode.ai/docs/mcp-servers/)

### Inferences
- **Best fit for OpenCode:** the Zowe MCP server (stdio and SSH, with tiered guardrails). It needs Node 22 and a tarball install until it reaches npm.
- **Zero-install alternative:** sites already licensed for IDzEE/ADFz can use the RSE API MCP, a server-side remote endpoint OpenCode reaches as a `remote` MCP.
- The Z Open Editor MCP is technically reachable by OpenCode over localhost HTTP, but only while a VS Code/Bob window hosts it. That makes it awkward for a headless terminal agent.
- Community z/OSMF servers should be treated as unaudited code holding mainframe credentials.

### Gaps
- No Db2 for z/OS-, CICS- or IMS-specific operational MCP servers from IBM were found. Nor were Rocket Software MCP servers, or any watsonx Code Assistant for Z MCP exposure.
- I could not get the full RSE API MCP tool list or the full Zowe MCP tool name list.

## Q4. Direct SSH to z/OS UNIX (USS), tsocmd/opercmd, z/OS Open Tools

### Takeaway
An agent can run over plain `ssh user@zos 'cmd'` from the workstation. Zowe's `zos-ssh` and the Zowe MCP use the same SSH channel. On USS, TSO and console commands are reachable via `tsocmd` and ZOAU's `opercmd`. Git, bash and other GNU-style tools come from z/OS Open Tools (zopen) or IBM Open Enterprise Foundation. The agent process itself (OpenCode, a Go/Bun binary) would not run natively on z/OS; it runs on the workstation and drives z/OS remotely.

### Cited Findings
- Zowe `zos-ssh` runs USS shell commands over SSH and needs the user to be permitted SSH login. Zowe 3.5's `--exec` skips shell init — [core groups](https://docs.zowe.org/stable/user-guide/cli-using-understanding-core-command-groups), [Zowe 3.5](https://openmainframeproject.org/blog/whats-new-in-zowe-3-5/)
- Zowe docs recommend OpenSSH on z/OS so management can happen via SSH instead of OMVS over TN3270 — [Zowe z/OS requirements](https://docs.zowe.org/stable/user-guide/systemrequirements-zos)
- zopen / z/OS Open Tools provides almost 300 open-source ports (bash, git, etc.). IBM Open Enterprise Foundation (OEF) also ships bash and others, with `ZOPEN_HOME` such as `/usr/lpp/IBM/foz/v1r1` for OEF — [IBM Community: Pimp my z/OS](https://community.ibm.com/community/user/blogs/mathieu-dalbin/2024/08/26/pimp-my-zos-enhancing-experience-with-unix-system) (2024, older); project docs at https://zosopentools.github.io/meta/ (returned 404 in this session)
- Older route: Rocket Git for z/OS, about 550 MB, tarball or SMP/E — [IBM Docs: Setting up Git on USS](https://www.ibm.com/docs/SS6T76_1.1.0/setup_git_on_uss.html) (older)
- (prior knowledge, not re-fetched)
  - `tsocmd` is a z/OS UNIX command that runs a TSO/E command from the shell — [IBM Docs: tsocmd](https://www.ibm.com/docs/en/zos/3.1.0?topic=descriptions-tsocmd-run-tsoe-command-from-shell).
  - `opercmd` and other utilities (`mls`, `dls`, `dcp`, `jsub`, `pjdd`, `decho`, `mvscmd`) come from IBM Z Open Automation Utilities (ZOAU), a prerequisite of the Ansible z/OS collection — [IBM Docs: ZOAU](https://www.ibm.com/docs/en/zoau).
  - Non-interactive SSH sessions often skip `/etc/profile`, so PATH and `_BPXK_AUTOCVT` may not be set (a community thread on bash in non-interactive SSH: [Rocket community](https://community.rocketsoftware.com/question/issue-with-bash-usage-during-non-interactive-ssh-session)).

### Inferences
- The SSH path is the most universal one. It needs only OpenSSH and a USS-enabled ID with an OMVS segment, and no z/OSMF.
- Agents should wrap commands as `ssh host '. /etc/profile; ...'` or explicitly export the environment.
- Running a full agent on z/OS is not realistic as of 2026. The OpenCode binary has no z/OS build (gap). The Zowe MCP "run on z/OS" PR only moves the MCP server, not the LLM agent, onto z/OS.

### Gaps
- I did not verify current zopen install commands (`zopen init`/`zopen install`), or Python availability via IBM Open Enterprise SDK for Python.
- I did not verify whether OpenCode or Bun has any z/OS port; it is assumed not.

## Q5. z/OSMF REST APIs via curl, and Ansible (ibm.ibm_zos_core) as an agent-callable layer

### Takeaway
z/OSMF REST (files, jobs, TSO, console, workflows) is what Zowe CLI wraps, and an agent can call it directly with curl. Ansible's `ibm.ibm_zos_core` (latest release 2.1.0) is a declarative, idempotent layer over SSH plus ZOAU and Python on z/OS. It suits repeatable change tasks better than ad-hoc exploration.

### Cited Findings
- (prior knowledge, not re-fetched)
  - **Endpoints:** `/zosmf/restfiles/ds`, `/zosmf/restfiles/ds/<dsn>(<member>)`, `/zosmf/restfiles/fs?path=`, `/zosmf/restjobs/jobs` (PUT to submit; GET `.../files/<id>/records` for spool), `/zosmf/tsoApp/tso`, `/zosmf/restconsoles/consoles/defcn` and `/zosmf/workflow/rest/1.0/workflows`.
  - **Headers:** requests need `X-CSRF-ZOSMF-HEADER: true` and Basic auth, a client cert or an LTPA/JWT token.
  - **Data transfer:** headers `X-IBM-Data-Type: text|binary|record` and `X-IBM-Data-Type: text;fileEncoding=IBM-037` control conversion.
  — [IBM Docs: z/OSMF REST services](https://www.ibm.com/docs/en/zos/3.1.0?topic=guide-zos-jobs-rest-interface)
- z/OSMF is the dependency of the `provisioning`, `zos-workflows` and `zosmf` Zowe groups — [core groups](https://docs.zowe.org/stable/user-guide/cli-using-understanding-core-command-groups)
- **ibm_zos_core functional areas:** batch jobs, JES/MVS commands, data set operations, RACF user management, volume init and started tasks — [GitHub ansible-collections/ibm_zos_core](https://github.com/ansible-collections/ibm_zos_core)
- **Latest release 2.1.0:**
  - New `zos_user_info` (RACF user/group profiles).
  - `zos_backup_restore` rename-on-restore.
  - `zos_stat` gains ISPF stats and member attributes.
  - Fixes to `zos_copy`, `zos_encode`, `zos_job_query`, `zos_job_submit` and `zos_script`.
  — [Release notes](https://ibm.github.io/z_ansible_collections_doc/ibm_zos_core/docs/source/release_notes.html)
- Earlier 2024–25 releases:
  - 1.16.0-beta.1 (Oct 2025) added `zos_started_task` and warned of aliases deprecated in 2.0.0 — [IBM Community](https://community.ibm.com/community/user/blogs/oscar-fernando-flores-garcia/2025/10/31/ibm-zos-ansible-core-collection-version-1160-beta)
  - 1.13.0: `zos_job_submit` with `wait_time_s: 0` (fire-and-forget) and async, with follow-up through `zos_job_query`/`zos_job_output` — [IBM Community](https://community.ibm.com/community/user/blogs/oscar-fernando-flores-garcia/2025/03/31/ibm-zos-core-collection-v1130-is-out-now-providing)
  - 1.12.0: `zos_operator` `case_sensitive` option — [IBM Community](https://community.ibm.com/community/user/blogs/oscar-fernando-flores-garcia/2024/12/09/ansible-core-v1120-is-out-and-supports-zos-31)
- The collection is also in the Red Hat certified catalog — [Red Hat catalog](https://catalog.redhat.com/software/collection/ibm/ibm_zos_core)
- (prior knowledge, not re-fetched) The managed-node requirements are SSH, IBM Open Enterprise SDK for Python and ZOAU, with exact versions on the software-requirements page — [requirements](https://ibm.github.io/z_ansible_collections_doc/ibm_zos_core/docs/source/requirements/software-requirements.html)

### Inferences
- Agent pattern: have OpenCode *write* Ansible playbooks (reviewable, idempotent, check-mode capable) and run `ansible-playbook --check` before a real run. This is a safer change-control story for sysprogs than free-form console commands.
- Raw curl to z/OSMF is a fallback when Zowe CLI is unavailable, at the cost of handling CSRF headers and encoding manually.

### Gaps
- No release date for 2.1.0 was shown, and I did not confirm the exact ZOAU and Python versions it requires.

## Q6. Language servers (COBOL, PL/I, HLASM, REXX, JCL) usable by OpenCode via LSP

### Takeaway
OpenCode supports custom stdio LSP servers via the `lsp` key (LSP is disabled by default). The best open candidates are Broadcom/Eclipse Che4z's COBOL Language Support and HLASM Language Support, both LSP-based and open source. IBM Z Open Editor's COBOL, PL/I, HLASM, REXX and JCL language servers ship inside a VS Code extension. Extracting them for OpenCode is unsupported and unverified.

### Cited Findings
- OpenCode `lsp` entries take `command` (array), `extensions`, `env`, `initialization` and `disabled`. "LSP is disabled by default". The built-in list has no COBOL, PL/I, HLASM, REXX or JCL servers — [OpenCode docs: LSP](https://opencode.ai/docs/lsp/)
- COBOL Language Support is an Eclipse Che4z open-source LSP project — [github.com/eclipse/che-che4z-lsp-for-cobol](https://github.com/eclipse/che-che4z-lsp-for-cobol), [README](https://cdn.jsdelivr.net/npm/@codingame/monaco-editor-wrapper@31.0.0/dist/assets/broadcomMFD.cobol-language-support-2.5.1.vsix/readme.md)
  - v2.3.0 added "Allow repeating analysis in CLI" — [Broadcom community](https://community.broadcom.com/discussion/cobol-language-support-version-230-is-now-available)
- Code4z includes HLASM Language Support (completion, highlighting, validation), and "Code4z extensions utilize language servers that are fully compatible with the Language Server Protocol" — [Eclipse projects](https://projects.eclipse.org/node/17525), [code4z](https://github.com/BroadcomMFD/code4z/pull/18/files)
- Z Open Editor (IBM Developer for z/OS on VS Code) provides editing for COBOL, PL/I, REXX, JCL and Assembler — [github.com/IBM/mcp](https://github.com/IBM/mcp)

### Inferences
- Plausible OpenCode setup: unpack the Che4z COBOL or HLASM `.vsix` and point `lsp.cobol.command` at the bundled server (a Java jar for COBOL LS; a native/WASM binary for HLASM LS), with `extensions: [".cbl", ".cob", ".cpy"]`. Copybook resolution would need to be configured (local folders or Zowe-downloaded copybooks).
- Z Open Editor's servers are proprietary and tied to the VS Code extension host.

### Gaps
- I did not verify that the Che4z COBOL or HLASM servers run standalone over stdio outside VS Code, or which launch arguments they need.
- No standalone JCL or REXX LSP was found.

## Q7. Codepage/EBCDIC, record format and other pitfalls when an agent edits mainframe data

### Takeaway
The main risks:
- Silent codepage mis-conversion (IBM-1047 vs IBM-037 vs site CCSIDs; `[ ] ^ ! |` differ).
- Fixed-length records: FB80 truncation or padding, and columns 73–80 sequence numbers in COBOL and JCL.
- Binary or packed data corrupted by text mode.
- Mismatched USS file tags.
- Destructive operations such as console commands and job cancels.

Use explicit transfer modes, tag-aware git config and server-side least privilege.

### Cited Findings
- Zowe download modes:
  - `--binary`: no conversion.
  - `--record`: no conversion, with the record length prepended (safe for RECFM V/VB and binary).
  - `--encoding`: convert using a named codepage.
  — [Zowe CLI strings](https://raw.githubusercontent.com/zowe/zowe-cli/master/packages/cli/src/zosfiles/-strings-/en.ts)
- USS auto-conversion of untagged files happens only when `_BPXK_AUTOCVT=ON`, and only between IBM-1047 and ISO8859-1 — [IBM LE reference: file tagging](https://www.ibm.com/support/knowledgecenter/en/SSLTBW_2.1.0/com.ibm.zos.v2r1.ceea300/filtag.htm); see also [IBM Docs: Enhanced ASCII, z/OS 3.2](https://www.ibm.com/docs/SSLTBW_3.2.0/com.ibm.zos.v3r2.bpxb200/setupa.htm)
- Git on z/OS needs file tags to match `.gitattributes` (e.g. `* text zos-working-tree-encoding=IBM-1047`), or it fails with "file is tagged, set corresponding zos-working-tree-encoding attribute or reset file tag" — [ZOSOpenTools/gitport](https://github.com/ZOSOpenTools/gitport), [Rocket community thread](https://community.rocketsoftware.com/rocket-open-source-languages-and-tools-for-z-os-22/fatal-can-t-add-hello-java-file-is-tagged-set-corresponding-zos-working-tree-encoding-attribute-or-reset-file-tag-27019). The attribute name has changed between ports; it conflicts with upstream `working-tree-encoding` — [gitport issue #21](https://github.com/ZOSOpenTools/gitport/issues/21)
- The Zowe MCP exposes `chtag` as a tool, which shows that tagging is a first-class concern — [zowe-mcp](https://github.com/zowe/zowe-mcp)
- Guardrails:
  - Zowe MCP docs state that "client hints and confirmation dialogs reduce accidents" while SAF/RACF, USS ACLs and scheduler exits are what enforce boundaries. They recommend a dedicated least-privilege SSH user, and tool output is marked untrusted against prompt injection — [zowe-mcp](https://github.com/zowe/zowe-mcp)
  - Zowe docs warn that security admins should control which console commands are available — [core groups](https://docs.zowe.org/stable/user-guide/cli-using-understanding-core-command-groups)
  - IBM advises against auto-approving Z Open Editor MCP tools that write to z/OS — [IBM Docs: agent mode](https://www.ibm.com/docs/en/developer-for-zos/17.0.x?topic=overview-agent-mode)

### Inferences
Practical agent rules for a system prompt or AGENTS.md:
1. Never exceed LRECL (usually 72 usable columns for COBOL and JCL), and don't let editors strip trailing blanks or convert tabs.
2. Preserve columns 73–80.
3. Use `--binary` or `--record` for load modules, packed-decimal or VB data.
4. State the site CCSID explicitly (e.g. `--encoding IBM-037` vs `1047`) and keep it consistent between upload and download.
5. Check DCB attributes (`zowe files list ds --attributes --rfj`) before writing.
6. Back up members before overwriting.
7. Restrict console, TSO and job-cancel tools with OpenCode `tools`/permission globs plus RACF.

### Gaps
- I did not find a 2025–26 authoritative "AI agent editing mainframe data" pitfalls guide. The rules above are synthesized from tool docs.
- I did not verify whether Zowe CLI 3.x defaults to IBM-1047 or a profile-level `encoding` property in this session.
