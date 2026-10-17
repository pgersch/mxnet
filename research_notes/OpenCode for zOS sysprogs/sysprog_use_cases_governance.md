# AI Coding Agents for z/OS Systems Programmers: Use Cases, Risks and Governance (as of Oct 2026)

Research note: about 17 search and fetch calls. Primary sources were IBM announcements and community blogs, the Open Mainframe Project, SHARE, SD Times (BMC survey) and CROZ (an IBM partner).

Source labels:
- [VENDOR] means vendor marketing or vendor-authored material.
- [PARTNER] means an IBM business partner, which is not independent.
- [PRACTITIONER/COMMUNITY] means a community or practitioner source.
- [SURVEY] means survey data.

The search turned up very little independent practitioner material, such as Reddit r/mainframe threads or blogs about sysprog-specific LLM use. That absence is itself a finding (see Gaps).

## 1. Which sysprog tasks are good fits for AI agents?

### Takeaway
The use cases that are documented and productized are mostly read and explain tasks:
- explaining JCL and REXX
- natural-language Q&A over IBM docs and Redbooks
- operational insight queries (OMEGAMON, System Automation, workload scheduling)
- converting JCL/REXX into Ansible
- onboarding and documentation
- overnight job-failure triage

The agentic "act" tasks (submitting jobs, issuing console or TSO commands) exist only in early-stage, confirmation-gated form. I found no source for AI-assisted SMP/E, PARMLIB comparison, RACF command generation or HLASM exits specifically for sysprogs.

### Cited Findings
**JCL explanation**
- watsonx Code Assistant for Z (WCA4Z) 2.1 added JCL Code Explanation [VENDOR]:
  - The user selects a JCL step, picks a simple or detailed explanation, and can insert the result as JCL comments or download it.
  - Target users are new z/OS users learning JCL and "veteran sysprogs encountering unfamiliar utilities."
  - It requires a connection to the fine-tuned LLM through the watsonx Code Assistant service on IBM Cloud.
  - The post states no accuracy caveats.
  - Aug 2024 post by Rich Larin, Mike Fulton and others — [IBM Community](https://community.ibm.com/community/user/blogs/rich-larin/2024/08/12/wca4z-jcl-explanation)
- WCA4Z Code Explanation supports COBOL, JCL, PL/I and REXX [VENDOR]. IBM later announced code generation and Assembler support for WCA4Z — [IBM announcement](https://www.ibm.com/new/announcements/ibm-watsonx-code-assistant-for-z-adds-ai-code-generation-and-assembler-support); WCA4Z 2.5 (Apr/May 2025) — [IBM Community](https://community.ibm.com/community/user/blogs/kelly-paesano/2025/04/30/ibm-watsonx-code-assistant-for-z-25-brings-enhance)

**JCL/REXX to Ansible**
- SHARE Kansas City 2024 session "Using Generative AI to Help Modernize System Programmers Tasks" [VENDOR-adjacent conference talk; speakers not listed on the page]:
  - It shows watsonx Code Assistant explaining JCL in natural language and then turning it into an Ansible playbook.
  - Its stated goal is to help early-tenure sysprogs build skills and work more independently, within Ansible-based enterprise automation.
  - [SHARE proceedings](https://www.share.org/Events/Past-Events/Proceedings/Proceeding-Details/using-generative-ai-to-help-modernize-system-programmers-tasks)
- Ansible Lightspeed with IBM watsonx Code Assistant generates full playbooks from natural-language prompts, single or multi-task (June 2024), in VS Code, and explains generated playbooks [VENDOR]. IBM positions Red Hat Ansible Automation Platform as the strategic automation choice for IBM Z — [IBM Community WCA4A](https://community.ibm.com/community/user/blogs/matt-rodkey/2024/06/25/wca4a-playbook-generation-and-onprem-availability); [Red Hat Lightspeed](https://developers.redhat.com/products/ansible/lightspeed). I found no confirmation of z/OS-collection-specific (ibm_zos_core) quality in Lightspeed.

**watsonx Assistant for Z (wxA4Z)**
- Its scope is sysprogs, operators and developers [VENDOR]:
  - conversational Q&A over IBM documentation and Redbooks (domain RAG)
  - running imported "trusted automation" (Ansible, JCL, REXX) as skills
  - up to 30 assistants per persona or department
  - [IBM Community GA post, Jun 2024](https://community.ibm.com/community/user/blogs/khadija-souissi/2024/06/19/ibm-watsonx-assistant-for-z-is-now-generally-avail); [SHARE partner listing](https://www.share.org/Partners/Partner-Directory/Partner-Details/get-conversational-answers-to-zos-questions-and-launch-automation-with-ibm-watsonx-assistant-for-zos); [IBM product page](https://www.ibm.com/products/watsonx-assistant-for-z)
- IBM's sysprog example scenario is checking and renewing an SSL certificate through chat [VENDOR, May 2025] — [IBM Community](https://community.ibm.com/community/user/blogs/sunayana-sharma/2025/05/29/a-mainframe-conversational-ai-assistant)
- CROZ partner write-up "Sysprog buddy" (Tomislav Mašić, 29 Jan 2025, updated through June 2025) [PARTNER, opinion rather than a measured trial]:
  - Use cases: platform Q&A from docs and Redbooks; a hypothetical Db2 table-creation scenario with naming, allocation and parameter guidance; running Ansible/JCL/REXX skills.
  - Operational queries such as LPAR health and Db2 buffer pools (v2.1).
  - Onboarding early-career staff.
  - [CROZ](https://croz.net/watsonx-assistant-for-z-sysprog-buddy/)

**Operational insight agents**
- System Insights agents in wxA4Z (Oct 24 2025, Wolfgang Schäberle) [VENDOR]:
  - OMEGAMON Insights Agent: health of LPARs, Db2, IMS, MQ, CICS, network, storage, JVMs and situation events.
  - Automation Insights Agent: IBM Z System Automation / NetView status, including Canzlog messages.
  - Workload Scheduler Insights Agent: upcoming or delayed batch jobs and critical jobs.
  - All of them work through tool calls via the ChatOps (AIOps integration) server.
  - The post does not mention SMF, RMF or Health Checker as direct data sources.
  - [IBM Community](https://community.ibm.com/community/user/blogs/wolfgang-schberle1/2025/10/24/system-insights-agents-in-wxa4z)

**General agents over Zowe MCP**
- GSUK Virtual Conference, 21–23 Apr 2026: Joe Winchester (IBM STSM, OMP Ambassador) demonstrated an AI agent connected to Zowe through an MCP server built on Zowe CLI [COMMUNITY / Open Mainframe Project]:
  - Demonstrated actions: listing datasets, viewing members, submitting jobs, and TSO/console commands.
  - Use cases discussed (not all demonstrated):
    - developer onboarding
    - overnight job-failure triage (read syslog, identify abends, suggest or execute fixes)
    - explaining legacy JCL/COBOL
    - compliance and audit Q&A on system state, job history and configuration
  - Page last updated 2026-07-01 — [Open Mainframe Project blog](https://openmainframeproject.org/blog/using-zowe-in-an-ai-world-with-mcp-agents/)
- Community MCP servers exist that wrap the z/OSMF REST API to browse datasets, read JCL/COBOL members and submit and monitor jobs, with a mock mode. There is also a community Endevor MCP server (43 tools, Endevor REST API v2). These are unverified community projects — [Glama mcp-mainframe](https://glama.ai/mcp/servers/naveenkrishnan01/mcp-mainframe); [Glama Endevor-MCP](https://glama.ai/mcp/servers/yzc1b7oe5f)
- A Broadcom architect's LinkedIn profile claims two things [individual's claim, not an official Broadcom announcement]:
  - a "Code4z Assistant" MCP server for Endevor (source, history, dependency analysis), in beta in January 2026 and in customer validation with large banks
  - work on an open-source Zowe MCP server (EPL-2.0) for data sets and batch jobs
  - [LinkedIn – Petr Plavjaník](https://cz.linkedin.com/in/plavjanik)

**What mainframe staff actually use AI for**
- BMC 2026 Mainframe Survey (>1,300 respondents, reported 1 Sep 2026) [SURVEY, vendor-run]: the common AI tasks are performance tuning, problem detection, IMS queue management and documentation generation — [SD Times](https://sdtimes.com/mainframe/2026-annual-bmc-survey/)
- The 2024 Arcati Mainframe Survey found about 25% of participants developing or deploying an AI/ML model on their mainframes [older, 2024] — [Planet Mainframe, Mar 2024](https://planetmainframe.com/2024/03/8-generative-ai-use-cases-for-the-mainframe/)

### Inferences
- **Strongest fits** (low blast radius, output verified by a human), ranked by the available evidence:
  1. Explaining existing JCL, REXX and CLIST code.
  2. Drafting JCL, REXX and Ansible for review.
  3. Docs Q&A and onboarding.
  4. Summarizing job output, syslog and OMEGAMON data.
  5. Drafting documentation and runbooks.
- PARMLIB/PROCLIB diffing, Health Checker and IPL output summarization, and SMF/RMF report interpretation follow the same read-and-summarize pattern. They are plausible fits, but no source documents them.
- A general agent such as OpenCode or Claude Code reaches z/OS mainly through Zowe CLI, z/OSMF REST or MCP servers. It is therefore naturally suited to file and text work on members pulled to the workstation, plus job submission and output retrieval.

### Gaps
- I found no sources specifically on AI for SMP/E (RECEIVE/APPLY/ACCEPT, HOLDDATA review), RACF/ACF2/Top Secret command generation, HLASM exits, USS/zFS administration, or z/OSMF workflow authoring.
- I found no quantified productivity or accuracy data for any sysprog task.

## 2. Which tasks are risky or poor fits, and what are the known failure modes?

### Takeaway
Vendor and community sources agree on one principle: read-only operations are safe to automate, while side-effecting ones (job submission, console and TSO commands, and above all destructive actions) need confirmation and human control. Concrete, documented failure cases for sysprog work, such as hallucinated parameters, wrong-release syntax or EBCDIC bugs, were not found in citable sources. That is a significant evidence gap.

### Cited Findings
- The Zowe MCP design [COMMUNITY]:
  - Tools are classified by risk. Read-only tools run without prompting, side-effecting operations (for example job submission) require confirmation, and destructive operations get additional checks.
  - The server is "not a raw connection to z/OS."
  - The project is at an early stage. Avoiding a "runaway process" is a stated design goal, and the community is "still determining where guardrails should be," with input sought from experienced sysprogs.
  - [OMP blog](https://openmainframeproject.org/blog/using-zowe-in-an-ai-world-with-mcp-agents/)
- Practitioner sentiment from the BMC 2026 survey [SURVEY]:
  - Code management: 40% are willing to let AI suggest actions, but only 23% would let AI complete them alone.
  - Database reorganization: 43% want AI recommendations (up from 37%), but only 21% would let AI finish unaided.
  - BMC's John McKenny says humans will remain in the loop.
  - [SD Times](https://sdtimes.com/mainframe/2026-annual-bmc-survey/)
- The CROZ partner piece says full functionality needs training and customization, and that human expertise remains necessary. The note writer infers further risks: answer quality depends on what documentation was ingested, and new staff may over-rely on answers — [CROZ](https://croz.net/watsonx-assistant-for-z-sysprog-buddy/)
- A general LLM failure pattern: hallucinations are fluent, authoritative and syntactically valid, so parsers do not reject them. For JCL that means a plausible but wrong parameter may surface only at runtime [general, not mainframe-specific] — [Coralogix](https://coralogix.com/ai-blog/llm-hallucinations/)
- A vendor of a TN3270-native agent argues that "a chatbot next to a terminal is not enough" for mainframe work [VENDOR claim] — [Hypercubic Hopper](https://hypercubic.ai/en/hopper)
- IBM's JCL explanation post states no accuracy caveats or verification advice. Vendor material generally omits error rates — [IBM Community](https://community.ibm.com/community/user/blogs/rich-larin/2024/08/12/wca4z-jcl-explanation)

### Inferences
- **High-risk or poor fits** (inferred from the confirmation-gating model and the low trust in the survey):
  - autonomous console commands on production LPARs
  - SMP/E APPLY/ACCEPT without human HOLDDATA review
  - RACF/ACF2/TSS changes such as PERMIT, ALTUSER and SETROPTS
  - PARMLIB changes activated via SET commands or at IPL
  - APF/LNKLST/LPA updates
  - HLASM exits that run in supervisor state
  - catalog and dataset deletes
  - zFS reformat or unmount
- **Likely failure modes**, inferred from general LLM behaviour and not documented in mainframe sources:
  - invented PARMLIB keywords
  - operands from the wrong z/OS release (for example 2.x versus 3.x)
  - wrong DCB/SPACE parameters
  - REXX code that works on Regina/ooRexx but not TSO/E REXX
  - EBCDIC/ASCII and codepage (IBM-1047) mistakes in USS file tagging or when transferring members through Zowe
  - line-length problems (columns 72–80 in JCL)

### Gaps
- I found no citable Reddit r/mainframe, IBM-MAIN or SHARE practitioner reports of specific LLM errors in sysprog work. A direct search for Reddit content returned nothing indexed.
- There are no published accuracy benchmarks for LLMs on JCL, REXX, SMP/E or RACF syntax.

## 3. Documented experiences, and IBM's AI offerings compared with general agents (OpenCode, Claude Code, Copilot)

### Takeaway
Most "experience" material is vendor or partner content. IBM's stack, by release:
- watsonx Assistant for Z (agentic since Oct 2025, v3.2/3.3, Spyre on-platform inference)
- WCA4Z for code explanation and generation
- System Insights agents over OMEGAMON, System Automation and Workload Scheduler
- z/OS 3.2 AI Framework

General agents complement this stack through Zowe/MCP access and broad code and text skills. They do not have IBM's tuned RAG over z/OS docs or its trusted-automation catalog.

### Cited Findings
**IBM watsonx Assistant for Z, by date**
- GA June 2024. It was originally built on watsonx Orchestrate with a custom granite.13b.labrador LLM; that model detail is likely dated — [IBM Community GA](https://community.ibm.com/community/user/blogs/khadija-souissi/2024/06/19/ibm-watsonx-assistant-for-z-is-now-generally-avail); [Futurum](https://futurumgroup.com/insights/ibm-unveils-watsonx-assistant-for-mainframe-mastery)
- Agentic release announced 7 Oct 2025 [VENDOR]:
  - an agentic AI framework with multi-agent orchestration
  - a no-code Agent Builder and a pro-code Agent Development Kit
  - example agents: OMEGAMON insights, ServiceNow incident response, accelerated Z upgrades
  - agents operate "with minimal supervision and added manual control when needed"
  - Spyre support announced as a statement of direction for 4Q 2025
  - [IBM announcement](https://www.ibm.com/new/announcements/ibm-watsonx-assistant-for-z-bringing-intelligence-and-automation-to-mainframe-it-management-with-agentic-ai)
- Secondary reports say the release supports Granite plus third-party and open-source LLMs, and reaches z/OS, Db2, CICS, IMS and MQ via MCP — [futurecio](https://futurecio.tech/new-agentic-ai-capabilities-in-ibm-z-launched); [IBM "Empowering IBM Z users with agentic AI"](https://www.ibm.com/new/announcements/empowering-mainframe-users-with-agentic-ai). The fetched IBM announcement page did not itself mention MCP, so treat the MCP claim as secondary.
- wxA4Z now runs with the Spyre Accelerator, with on-platform inference to keep data private — [IBM announcement](https://www.ibm.com/new/announcements/watsonx-assistant-for-z-now-powered-by-spyre-accelerator). Documentation exists for v3.3.0, including a 3.2 to 3.3.0 upgrade guide — [IBM Docs](https://www.ibm.com/docs/en/watsonx/waz/3.3.0)
- Deployment evolution per CROZ [PARTNER]:
  - v1 was on-prem only with heavy hardware; later versions support lower-cost GPUs such as the L40S
  - a dedicated per-customer cloud instance is available
  - v2.2 added Linux on Z (s390x) installation, with the LLM on x86
  - v2.2.5 expanded PII support
  - role, team and access-level customization
  - [CROZ](https://croz.net/watsonx-assistant-for-z-sysprog-buddy/)

**z/OS 3.2**
- Announced 22 Jul 2025, GA 30 Sep 2025, with z17 support. It is positioned as built for hybrid cloud and AI, with hardware-accelerated AI and operational AI insights — [ADMIN Magazine](https://www.admin-magazine.com/News/IBM-Z-Announces-z-OS-3.2)
- The 2Q 2026 Continuous Delivery drop added [VENDOR]:
  - AI Framework for z/OS support for MLz 3.2.0.26 and SAF authorization (access control and auditability)
  - AI-powered OSA outbound batching for OSH (PTF PH69692)
  - zCX support for up to 10 image registries and removal of the 1 TB guest memory limit
  - expected in RSU2609
  - The post mentions no z/OSMF AI assistant or MCP/agent features.
  - [IBM Community, Jun 2026](https://community.ibm.com/community/user/blogs/sneha-kanaujia1/2026/06/05/designed-for-ai-simpler-by-default)

**Other vendors**
- BMC announced "governed AI agents" for mainframe operations — [BMC newsroom](https://www.bmc.com/newsroom/releases/bmc-brings-governed-ai-agents-to-enterprise-workflows-and-mainframe-operations.html)
- Broadcom positions OPS/MVS within WatchTower for mainframe AIOps (policy and AI-driven automation). I found no Broadcom sysprog agent or MCP interface for OPS/MVS — [SoftwareOne listing](https://platform.softwareone.com/product/opsmvs-event-management-automation/PCP-4303-3735)
- The BMC 2026 survey found 36% of organizations plan to build their own AI agents and 32% plan to buy third-party ones — [SD Times](https://sdtimes.com/mainframe/2026-annual-bmc-survey/)

**GitHub Copilot**
- Copilot material is COBOL/application-modernization focused (business-logic analysis, file inventories) rather than sysprog work — [GitHub Blog](https://github.blog/ai-and-ml/github-copilot/how-github-copilot-and-ai-agents-are-saving-legacy-systems/)

### Inferences
- **Complement:** a general agent such as OpenCode or Claude Code (or Copilot agent mode) can serve as the sysprog's workstation-side tool:
  - editing JCL, REXX and Ansible in git
  - reviewing diffs of PARMLIB members exported via Zowe
  - drafting z/OSMF workflow XML and runbooks
  - driving Zowe CLI and MCP in read-only mode
- IBM wxA4Z would remain the governed, on-platform channel for operational actions, with its trusted skills and OMEGAMON/System Automation integration.
- **Compete:** both now expose MCP-style tool use. Model-agnostic agents such as OpenCode can point at self-hosted models, which overlaps with wxA4Z's "bring your own LLM" agent builder.

### Gaps
- I found no independent head-to-head evaluation of wxA4Z or WCA4Z against general agents for sysprog tasks.
- I found no IBM/Rocket statement specifically on OpenCode, or Claude Code against z/OS.
- Rocket Software sysprog AI material was not found in this pass.

## 4. Guardrails: least-privilege IDs, sandboxes, change control, audit, data protection, on-prem models, human approval

### Takeaway
The emerging consensus guardrails are:
- start read-only
- risk-tier the tools (read: automatic; write: confirm; destructive: extra checks)
- enforce limits at the backend, not just in the agent
- keep a human in the loop
- prefer on-platform or on-prem inference for sensitive data

RACF-specific agent-ID guidance was not found. Mainframe-adjacent IBM i MCP guidance is the closest analog.

### Cited Findings
- Zowe MCP risk tiers: read-only tools run automatically, job submission needs confirmation, and destructive operations get extra checks. The server is not a raw z/OS connection — [OMP blog](https://openmainframeproject.org/blog/using-zowe-in-an-ai-world-with-mcp-agents/)
- A read-only label at the MCP layer "is not the final security boundary." Enforce read-only both at the MCP layer and at the backend with a dedicated minimal-privilege role, and test that prohibited writes fail [general MCP guidance] — [Techstrong.ai](https://techstrong.ai/features/why-read-only-mcp-is-not-read-only-until-the-database-agrees/)
- IBM's IBM i MCP server security guidance: a dedicated user profile with narrow object authorities, no special authorities (no *ALLOBJ), and tool filtering on read-only hints. This is the closest platform analog to a dedicated RACF ID without SPECIAL/OPERATIONS — [Glama mirror of IBM/ibmi-mcp security doc](https://glama.ai/mcp/servers/@IBM/ibmi-mcp/blob/bdd94b498101779611bdbd43d33d837897cfed11/docs/agents/security.mdx)
- Read-only is "often the only defensible posture until tool permissions, consent propagation, and audit trails are mature" — [NHI Mgmt Group](https://nhimg.org/faq/when-is-read-only-access-better-than-write-access-for-mcp-servers/)
- On z/OS itself, AI Framework for z/OS gained SAF authorization for access control and auditability (2Q 2026 CD) — [IBM Community](https://community.ibm.com/community/user/blogs/sneha-kanaujia1/2026/06/05/designed-for-ai-simpler-by-default)
- wxA4Z governance features [VENDOR/PARTNER]:
  - authorized users trigger only trusted workflows — [IBM announcement](https://www.ibm.com/new/announcements/ibm-watsonx-assistant-for-z-bringing-intelligence-and-automation-to-mainframe-it-management-with-agentic-ai)
  - role- and access-level customization, PII handling (v2.2.5), and an on-prem preference because internal docs are sensitive — [CROZ](https://croz.net/watsonx-assistant-for-z-sysprog-buddy/)
  - Spyre on-platform inference so data stays on IBM Z — [IBM](https://www.ibm.com/new/announcements/watsonx-assistant-for-z-now-powered-by-spyre-accelerator)
- Regulated-industry constraints:
  - BMC 2026 survey: security and privacy is a major barrier for 39% and regulatory/compliance for 22%; respondents want AI as an advisor, not in full control — [SD Times](https://sdtimes.com/mainframe/2026-annual-bmc-survey/)
  - KPMG Q1 2026 banking pulse: data security, privacy and regulatory risk are the defining constraints on scaling AI — [KPMG PDF](https://kpmg.com/kpmg-us/content/dam/kpmg/pdf/2026/ai-quarterly-pulse-survey-banking-q1-2026.pdf)
  - A Tech Mahindra study cited in trade press found only 1 in 5 banks have robust real-time data governance — [CFOtech](https://cfotech.news/story/banks-double-ai-budgets-but-lag-behind-in-data-governance)
- The BMC survey also flags TLS certificate lifecycle shortening (by 2029) and finds 25% of shops still managing certificates manually. This is relevant because certificate renewal is IBM's showcase wxA4Z sysprog scenario — [SD Times](https://sdtimes.com/mainframe/2026-annual-bmc-survey/); [IBM Community](https://community.ibm.com/community/user/blogs/sunayana-sharma/2025/05/29/a-mainframe-conversational-ai-assistant)

### Inferences
These are recommended practices synthesized from the cited patterns, not taken from a single source.

**Agent identity and privileges**
- Give the agent a dedicated RACF/ACF2/TSS ID, ideally a PROTECTED ID used through Zowe/z/OSMF with MFA or passtickets.
- Do not give it SPECIAL, OPERATIONS or AUDITOR.
- Give it READ-only access to PARMLIB, PROCLIB and SMP/E CSIs.
- Allow no OPERCMDS profiles beyond display commands (D *), and no UPDATE to APF or LNKLST libraries.

**Where it runs**
- Run the agent first against a sandbox: ZD&T, Wazi as a Service, or a test/sandbox LPAR.
- Promote agent-drafted changes only through existing change management, with git plus a CAB ticket.
- Keep SMP/E APPLY CHECK, and human review of ++HOLD data, mandatory.

**Audit and data handling**
- Log agent activity with SMF type 80 (RACF) and type 30, and keep the agent's tool-call transcripts.
- Keep secrets, passwords, customer PII and production dataset contents out of cloud prompts.
- In banking and government, use self-hosted models: OpenCode supports local or OpenAI-compatible endpoints, and Claude is available via Bedrock or Vertex in-region; alternatively use wxA4Z on Spyre.

### Gaps
- None of the retrieved sources describes ZD&T or Wazi as a Service as an AI-agent sandbox. They are well-known IBM products, but no citation was gathered in this pass.
- I found no published RACF profile templates for AI agents, and no SHARE/GSE talk specifically on agent least-privilege on z/OS.
- I found no regulator guidance (for example DORA, FFIEC or OCC) that specifically addresses AI agents on mainframes.
