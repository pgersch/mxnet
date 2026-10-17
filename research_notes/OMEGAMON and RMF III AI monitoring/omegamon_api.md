# Programmatic API access to IBM OMEGAMON (z/OS) monitoring data for an interval-polling AI agent

Research date: 2026-10-17. About 20 searches and fetches. IBM Docs pages often load only as overview stubs, so some details, such as the full OpenAPI path list, could not be retrieved and are listed under Gaps.

## 1. What APIs exist today, and which to choose

### Takeaway
There are four main programmatic routes today.
- **TEMS REST services (`/api/v1/...` on the z/OS hub or remote TEMS).** This is the best fit for a polling agent. It is pull-based JSON and covers real-time attributes, near-term history, situation status, and situation start/stop.
- **IBM Z OMEGAMON Data Provider (ODP).** This is the best fit for streaming. It pushes JSON to Kafka, TCP JSON Lines, HTTP/Splunk HEC and OTLP, and it serves a Prometheus scrape endpoint. It can also stream situation status changes.
- **Legacy ITM SOAP (CT_Get and related methods).** This still works, but the REST services replace it.
- **The newer Grafana stack.** This is the OMEGAMON Data Requester / Web UI 2.1 / OMEGAMON Gateway in TMS on z/OS 6.3.4 and 6.4. It is built on the REST API.

I found no IBM-shipped MCP server for OMEGAMON. IBM's AI integration is AI Insights 2.2 plus the watsonx Assistant for Z AIOps skills.

### Cited Findings
- **TEMS REST services: what it is.**
  - It is a REST API on the z/OS TEMS (both HUB and RTEMS) that serves OMEGAMON metrics as JSON. It "works just like ITM SOAP, but less complicated" and is implemented to OpenAPI Specification 3.0.3. — [IBM Community, J. Winterton, Sep 2023](https://community.ibm.com/community/user/blogs/joseph-winterton1/2023/09/08/omegamon-rest-api-now-ready-for-your-use)
  - It is available only on z/OS hub and remote monitoring servers, not on distributed TEMS. It reads real-time data and near-term history from the persistent data store (PDS), but not the Tivoli Data Warehouse. — [IBM Docs: How to specify requests](https://www.ibm.com/docs/en/SSAUBV/rest_api_how_to.html); [IBM Docs: Accessing data](https://www.ibm.com/docs/en/SSAUBV/rest_api_access_data.html)
  - The overview says v1 provides GET, POST, PATCH, PUT and DELETE endpoints. The 2023 launch was GET-only. — [IBM Docs: Accessing data](https://www.ibm.com/docs/en/SSAUBV/rest_api_access_data.html)
  - The service needs no extra installation. It uses the site's well-known Tivoli Management Services HTTP/HTTPS port on z/OS. — [IBM Docs: Accessing data](https://www.ibm.com/docs/en/SSAUBV/rest_api_access_data.html)
- **TEMS REST services: APARs/PTFs.**
  - OA64983 / UJ93639 (KLV).
  - OA65203 / UJ93638 (KDS).
  - OA65553 / UJ93910 (KDS) and UJ93911 (KLV): REST situation support.

  — [IBM Community 2023](https://community.ibm.com/community/user/blogs/joseph-winterton1/2023/09/08/omegamon-rest-api-now-ready-for-your-use)
- **TEMS REST services: enhancements.**
  - OA66384 / UJ95277: Take Action definitions, plus `originnode` and `atomize` filters on `/situations/status`.
  - OA66540–OA66595 / UJ95435–UJ95436: start/stop situations, Affinity ID filter, `cols=*` on `/api/v1/data`, RTEMS events returned from a hub `/situations/status` call, and SAMP/ATOM properties in table and column definitions.

  — [IBM Community, Jul 2024](https://community.ibm.com/community/user/blogs/joseph-winterton1/2024/07/02/omegamon-restapi-enhancements-are-ready-for-use)
- **ODP: architecture.**
  - Two components:
    - **OMEGAMON Data Broker:** a plug-in for the Zowe cross-memory server. Use either the server bundled with ODP or an existing Zowe install. Typically one per LPAR.
    - **OMEGAMON Data Connect:** a Java application that runs on or off z/OS. Many Data Brokers can feed one Data Connect.
  - Start order is Data Connect, then Data Broker, then the data source.

  — [IBM Docs: Getting started with ODP 1.1](https://www.ibm.com/docs/en/SST5X1E_1.1.0/kay-getting-started.html)
- **ODP: outputs and sources.**
  - Outputs:
    - JSON Lines over TCP (push)
    - HTTP POST JSON, including the Splunk HEC event endpoint (push)
    - Kafka topics with JSON (push)
    - OTLP/gRPC, experimental (push)
    - Prometheus metrics HTTP endpoint (pull/scrape)
  - Sources: OMEGAMON agents on z/OS, Db2 Automation Expert, Db2 Log Analysis Tool and Db2 Query Monitor.

  — [IBM Docs: ODP 1.1 intro](https://www.ibm.com/docs/en/SST5X1E_1.1.0/kay-intro.html)
- **ODP: Prometheus endpoint paths.** Data Connect exposes Prometheus at `/metrics`. Since the 2025 Q2 maintenance (APAR OA67938, PTF UJ97499), it can also expose multiple named endpoints at `/metrics/<endpoint_name>`. The same maintenance lets you split `connect.yaml` into several files and override parameters with environment variables. — [IBM APAR OA67938](https://www.ibm.com/support/pages/apar/OA67938)
- **ODP: situation status streaming.** The TEMS forwards situation status changes (start, stop, true, no longer true) to ODP as JSON, and ODP sends them on to Instana, Splunk, Elastic, Kafka, OTel and Prometheus/Grafana. — [IBM Docs: ODP (shared config)](https://www.ibm.com/docs/en/SSAUBV/zcommonconfig/odp.html)
- **ITM SOAP methods.**
  - CT_Get: retrieves real-time or historical agent data.
  - CT_Alert / CT_Reset / CT_Acknowledge / CT_Resurface: work with events.
  - CT_Activate / CT_Deactivate: start and stop situations or policies.
  - CT_Execute, CT_Redirect, CT_WTO, CT_Export and CT_EMail. CT_EMail is not available on z/OS, and CT_Export can't write to a file on z/OS.

  — [IBM Docs: ITM 6.3 SOAP methods](https://www.ibm.com/docs/SS3JRN_7.2.1/com.ibm.itm.doc_6.3/adminuse/soapmethods.htm)
- **ITM SOAP endpoint.** The default SOAP server port is 1920 and the path is `///cms/soap`. This comes from the IBM System Automation docs, seen in a search snippet only. — [IBM Docs: SOAP SERVER policy item](https://www.ibm.com/docs/en/SSWRCJ_4.3.0/DefineAutoPolicy/SOAP_SERVER_Policy_Item.html)
- **OMEGAMON Data Requester 1.1.** Introduced with TMS on z/OS 6.3.4, it is a Grafana plug-in that connects to Grafana via the OMEGAMON REST API.
  - It covers OMEGAMON AI for Db2, CICS, z/OS, Storage and JVM, plus OMEGAMON for Messaging.
  - E3270UI and TEP workspaces can be exported to Grafana.
  - It is ordered through Shopz and downloaded from Fix Central. The docs are at ibm.com/docs/en/zodr.

  — [IBM Community, Jun 2025](https://community.ibm.com/community/user/blogs/ramanath-shanbhag/2025/06/20/introducing-ibm-z-omegamon-data-requester-bridging)
- **IBM Z Monitoring Suite 2.3 (Nov 2025).**
  - TMS on z/OS 6.4 adds IBM Z OMEGAMON Web UI 2.1, which includes an OMEGAMON Gateway linking TEMS and Grafana, an Admin UI and the Data Requester.
  - It also adds Grafana OSS 12.1.1 as an ISO, and OMEGAMON AI Insights 2.2.
  - AI Insights 2.2 builds ML seasonal anomaly-prediction models and shows them in Grafana. It is containerized and runs on Docker for s390x or amd64.
  - Db2 Batch Reporter data can now stream through ODP.
  - Current agent versions: OMEGAMON AI for z/OS 6.1, AI for CICS 6.1, AI for Db2 6.1, AI for Networks 6.1, and OMEGAMON AI for JVM 6.1.

  — [IBM Docs: What's new in Z Monitoring Suite 2.3](https://www.ibm.com/docs/SSRLTD_2.3.0/whats_new_v23.html)
- **Other AI integrations.**
  - The OMEGAMON AI for z/OS product page says AI Insights builds baselines from historical data and predicts and alerts on abnormal behavior. — [IBM product page](https://www.ibm.com/products/z-omegamon-ai-zos)
  - IBM Z Integration for Observability (a separate offering) streams OMEGAMON metrics to Instana, Concert for Z, Grafana, Elastic, Splunk, Kafka and Prometheus. — [IBM product page](https://www.ibm.com/products/z-integration-observability)
  - watsonx Assistant for Z has a "Z AIOps integration server", a container on OpenShift. Its first AIOps skills use OMEGAMON data to answer z/OS health questions. — [IBM Community, Jan 2025](https://community.ibm.com/community/user/blogs/wolfgang-schberle1/2025/01/27/ibm-watsonx-assistant-for-z-how-to-leverage-z-aiop)
  - IBM uses MCP elsewhere on Z: the watsonx Assistant for Z CICS agents call a CICS MCP server. — [IBM Community, Nov 2025](https://community.ibm.com/community/user/blogs/kye-maloy/2025/11/14/demystifying-ibm-cics-transaction-server-agents)
  - IBM's MCP catalog on GitHub ([github.com/IBM/mcp](https://github.com/IBM/mcp)) showed no OMEGAMON entry in the search results.
- **Enhanced 3270UI (E3270UI).** It is an interactive 3270 interface, not an API. It uses its own SAF class (`RTE_SECURITY_CLASS`), separate from REST. — [IBM Docs: REST authorization](https://www.ibm.com/docs/en/SSAUBV/rest_api_authorization.html)

### Inferences
- **For an AI agent polling on an interval, use TEMS REST services.** It needs no new address spaces, ships as PTFs in TMS on z/OS 6.3.x, and returns JSON. It also exposes both raw attributes (`/data`) and situation status (`/situations/status`), so the agent can use OMEGAMON situations as ready-made thresholds.
- **ODP is better for continuous high-volume telemetry** into a time-series store. An agent would then query Prometheus, Kafka or Elastic rather than z/OS. The Prometheus pull endpoint is the natural fit if the agent already speaks PromQL.
- **Treat ITM SOAP as legacy.** IBM positions REST as its simpler replacement. Use SOAP only where the REST PTFs are not installed.
- **Avoid screen-scraping the E3270UI.**
- **No first-party MCP server exists**, so an agent would wrap TEMS REST calls (or PromQL against ODP data) in its own tools.

### Gaps
- I could not confirm whether the Data Requester or Gateway in TMS 6.4 exposes its own separate REST API for third parties.
- I found no IBM statement formally deprecating ITM SOAP on z/OS.
- I could not verify any OMEGAMON-specific MCP server, from IBM or the community, as of Oct 2026.
- I could not open the "OMEGAMON Integration Monitor" or the IBM Z Monitoring Suite licensing pages.

## 2. Endpoints, authentication, request/response formats, attribute tables

### Takeaway
TEMS REST:
- Base URL is `https://<tems-host>:<TMS HTTP(S) port>/api/v1`.
- Known paths are `/token`, `/timenow`, `/spec`, `/data`, `/situations/status` and `/situations/status/history`, plus `/system` managed-system paths.
- Auth is HTTP Basic or a Bearer token from `GET /token` (24h default lifetime).
- Authorization is via SAF class `$KOBSEC`.
- The full path and parameter list is served live as OpenAPI YAML at `GET /api/v1/spec`. An agent should load this at startup.

ODP produces JSON records keyed by `product_code` and `table_name`.

### Cited Findings
- **Base path and documented endpoints.** The base path is `https://host:port/api/version`, with version v1. Documented endpoints:
  - `GET /data` for collection data. With no time parameters it returns real-time data; with `timeFrom` it returns near-term history.
  - `GET /situations/status/history`, which requires `timeFrom`.
  - `GET /timenow`.
  - Managed-system requests include `/system` in the path.

  — [IBM Docs: How to specify requests](https://www.ibm.com/docs/en/SSAUBV/rest_api_how_to.html)
- **Time parameters and format.** `timeFrom` and `timeTill` use the 16-digit format `CYYMMDDHHMMSS000`, where the first three digits are years since 1900. `timeTill` defaults to now. Example request and response:
  - `GET https://host:port/api/v1/timenow`
  - `{"timenow": "1230717063437000"}`, which is 2023-07-17 06:34:37.

  — [IBM Docs: How to specify requests](https://www.ibm.com/docs/en/SSAUBV/rest_api_how_to.html)
- **OpenAPI document.** `GET https://host:port/api/v1/spec` returns the OpenAPI document (YAML) with all request paths, query parameters and responses. — [IBM Docs: OpenAPI document](https://www.ibm.com/docs/en/SSAUBV/rest_api_access_doc.html)
- **`/data` parameters.**
  - `cols=*` returns all columns (after OA66540-series PTFs).
  - `/situations/status` supports `originnode`, `atomize` (display item), `name` and `expand=true`.
  - `expand=true` requires `name` and returns history for running situations only.

  — [IBM Community, Jul 2024](https://community.ibm.com/community/user/blogs/joseph-winterton1/2024/07/02/omegamon-restapi-enhancements-are-ready-for-use)
- **Bearer token authentication.**
  - Get a token with `curl -u user:password https://host:port/api/v1/token`.
  - Use it with `curl -H "Authorization: Bearer <token>" https://host:port/api/v1/timenow`.
  - Tokens expire after 86,400 s (24 h) by default.

  — [IBM Docs: TEMS REST authentication](https://www.ibm.com/docs/en/SSAUBV/rest_api_authentication.html)
- **Basic authentication.**
  - Send `Authorization: Basic <base64 user:password>` with a mainframe user ID and password or passphrase. PassTickets and MFA are supported.
  - A request without an Authorization header returns HTTP 400.
  - Set `RTE_SECURITY_FOLD_PASSWORD_FLAG=N` for mixed-case passwords.
  - HTTPS is recommended.

  — [IBM Docs: TEMS REST authentication](https://www.ibm.com/docs/en/SSAUBV/rest_api_authentication.html); [IBM Docs: Accessing data](https://www.ibm.com/docs/en/SSAUBV/rest_api_access_data.html)
- **Authorization (SAF).**
  - Authorization uses SAF general resource class `$KOBSEC`. The class name is fixed and cannot be changed.
  - `KDS_TEMS_SECURITY_KDS_VALIDATE=Y` enables user ID and password validation. `RTE_SECURITY_USER_LOGON` names the security system.
  - Profiles cover four categories: collection data (real-time or historical), history collection configuration, situations, and Take Action. Mapped IDs connect to OPERCMDS profiles.

  — [IBM Docs: TEMS REST authorization](https://www.ibm.com/docs/en/SSAUBV/rest_api_authorization.html)
- **ODP situation-status record fields.**
  - Product code `ksm`.
  - Fields: `product_code`, `table_name`, `table_version`, `situation_name`, `predicate` (the formula), `type` (Pure, Sampled or Error), `write_time`, `system_name`, `sysplex_name`, `smf_id`, `local_timestamp` (16-digit), `deltastat` (Started, Stopped, True, No longer true or Error), `situation_results` (only when True) and `originnode`.
  - Enable it with `KDS_STREAM_SITUATION_STATUS=Y` for all situations; you cannot pick specific ones. This needs a TEMS restart and must be configured with Configuration Manager.

  — [IBM Docs: ODP shared config](https://www.ibm.com/docs/en/SSAUBV/zcommonconfig/odp.html)
- **ITM SOAP CT_Get request.**
  - Example body: `<CT_Get><hub>…</hub><object>NT_System</object><target>Primary:…:NT</target><userid>…</userid><password>…</password><history>Y</history><attribute>…</attribute><afilter>Write_Time;GT;1020804</afilter></CT_Get>`.
  - `object` is the attribute group and is required. `target` is the managed system and defaults to all. `attribute` can be repeated.

  — [IBM Docs: SOAP methods](https://www.ibm.com/docs/SS3JRN_7.2.1/com.ibm.itm.doc_6.3/adminuse/soapmethods.htm)
- **TEPS "ITM REST API".** This is a different interface: the TEPS/APM REST on port 15200/15210, under the path `/providers/itm.TEMS/datasources/<agent>/datasets`. It is not officially supported by IBM. — [rafal-szypulka/itm_exporter README](https://github.com/rafal-szypulka/itm_exporter)

### Inferences
- **Agent bootstrap sequence:**
  1. `GET /token` with Basic auth to get a token.
  2. Cache the token for less than 24 h and refresh it on HTTP 401.
  3. `GET /spec` to discover paths and parameters.
  4. Poll `/data` (with table and managed-system parameters as defined in the spec) and `/situations/status`.
- **Service account.** Use a dedicated RACF ID with READ-only `$KOBSEC` profiles for collection data and situations, and no Take Action or start/stop access, so the agent stays read-only.
- **Two different "REST APIs".** Do not confuse z/OS TEMS REST (`/api/v1`, supported) with the TEPS REST API (`/ibm/tivoli/rest/...`), which is undocumented and unsupported.

### Gaps
- **Exact `/data` query syntax** (product code, table, managed system, filter parameter names) and the JSON response shape: IBM Docs only point to the live `/spec`, and I did not find a public example. The 2026 docs index at ibm.com/docs/en/om-shared ("History data collection requests" topic) may have more.
- **Table names for KM5 (z/OS), CP (CICS), D5 (Db2) and MQ** were not retrieved. IBM points to the "OMEGAMON monitoring agents supported by ODP" topic and each agent's attribute reference. I did not verify specific table names, so none are given here.
- **Default HTTP/HTTPS port numbers** for TMS on z/OS are site-defined in the docs. Commonly cited values (1920 for HTTP, 3661 for HTTPS) come from ITM convention; only the SOAP port 1920 appeared in a search snippet, and I did not verify these for REST.
- **ODP Data Broker/Connect ports and `connect.yaml` / collection configuration keys** were not retrieved.

## 3. Situations and alerts vs raw attributes

### Takeaway
OMEGAMON situations are TEMS-evaluated threshold predicates. The agent can read the current situation state through REST `/situations/status`, which can be filtered and works across hub and RTEMS. It can read past state through `/situations/status/history` (needs `timeFrom`), or receive status changes as a push through ODP's `ksm` stream. Raw metrics come from `/data` (real-time, or PDS history with `timeFrom`).

### Cited Findings
- **Situation status endpoints.**
  - `/situations/status` on a hub also returns RTEMS events.
  - `expand=true&name=<sit>` returns history for running situations only.
  - Results can be filtered by `originnode` and `atomize`.
  - Situations can be started and stopped via REST.

  — [IBM Community, Jul 2024](https://community.ibm.com/community/user/blogs/joseph-winterton1/2024/07/02/omegamon-restapi-enhancements-are-ready-for-use)
- **Situation history.** `GET /situations/status/history` requires `timeFrom`. — [IBM Docs: How to specify requests](https://www.ibm.com/docs/en/SSAUBV/rest_api_how_to.html)
- **Streaming status changes.**
  - ODP streams situation status changes: Started, Stopped, True, No longer true and Error.
  - `situation_results` carries the triggering row data when the status is True.
  - Situation types are Pure (event-driven) or Sampled (re-evaluated on an interval).

  — [IBM Docs: ODP shared config](https://www.ibm.com/docs/en/SSAUBV/zcommonconfig/odp.html)
- **AI-based problem detection.** AI Insights 2.2 adds ML anomaly prediction on seasonal KPI trends, shown in Grafana. — [IBM Docs: Z Monitoring Suite 2.3](https://www.ibm.com/docs/SSRLTD_2.3.0/whats_new_v23.html)

### Inferences
- **Recommended problem-detection pattern:**
  1. Poll `/situations/status` each cycle. This is cheap and already thresholded, giving the "what is firing" view.
  2. For situations that are True, fetch the relevant `/data` table for the `originnode` to diagnose.
  3. Optionally pull `/data` with a short `timeFrom` window for trend context.
- **Push instead of poll.** If ODP is installed, subscribing to the `ksm` Kafka topic avoids polling for alerts altogether.

### Gaps
- The JSON response schema for `/situations/status` was not available publicly.
- I could not confirm whether REST exposes acknowledgement state, as SOAP's CT_Acknowledge does.

## 4. Rate, overhead and polling intervals

### Takeaway
IBM gives no published REST rate limit. Reasonable guidance:
- Poll at no faster than the agents' own sampling or situation intervals, with 60 s or more as a practical floor.
- Reuse bearer tokens rather than re-authenticating each time.
- Request only the needed columns rather than `cols=*`.
- Use ODP for high-frequency, wide data.

### Cited Findings
- **Token reuse.** IBM says bearer tokens let a session be reused across multiple requests, which "can improve response time" and reduces how often credentials are sent. — [IBM Docs: TEMS REST authentication](https://www.ibm.com/docs/en/SSAUBV/rest_api_authentication.html)
- **Community scrape guidance.** The ITM Prometheus exporter does not recommend scrape intervals under 60 s. It also says to include ORIGINNODE and the key attributes as labels to avoid duplicate series. — [itm_exporter](https://github.com/rafal-szypulka/itm_exporter)
- **General tuning reference.** The OMEGAMON Best Practices & Performance Guidelines (2020) is IBM's general tuning reference. I did not fetch it. — [IBM PDF](https://www.ibm.com/support/pages/sites/default/files/inline-files/$FILE/OMEGAMON%20Best%20Practices%20and%20Performance_Guidelines_V1%202020_0.pdf)

### Inferences
- **Each REST `/data` call is an on-demand TEMS-to-agent query**, comparable to opening a TEP or E3270 workspace. Wide fan-out (all managed systems, all columns) at short intervals adds TEMS and agent CPU. Prefer hub `/situations/status` plus targeted `/data` queries.
- **History queries need PDS history collection configured** for the table, so `timeFrom` returns nothing otherwise.

### Gaps
- I found no IBM-published REST throttling limits, recommended poll intervals, or CPU-cost figures.

## 5. Versions and licensing

### Takeaway
- **TEMS REST** needs TMS on z/OS (KDS/KLV) with the APARs listed in section 1, from 2023 and 2024. It ships at no extra charge as part of TMS.
- **ODP** is "IBM Z OMEGAMON Data Provider 1.1", with 2025 maintenance OA67938.
- **Data Requester** needs TMS on z/OS 6.3.4.
- **Web UI 2.1, Grafana and AI Insights 2.2** need TMS on z/OS 6.4 (Z Monitoring Suite 2.3, Nov 2025).
- **Current agents** are the "OMEGAMON AI for …" 6.1 family.

### Cited Findings
- **REST APARs.** OA64983/OA65203 (base), OA65553 (situations), OA66384 and OA66540–OA66595 (enhancements). — [Community 2023](https://community.ibm.com/community/user/blogs/joseph-winterton1/2023/09/08/omegamon-rest-api-now-ready-for-your-use); [Community 2024](https://community.ibm.com/community/user/blogs/joseph-winterton1/2024/07/02/omegamon-restapi-enhancements-are-ready-for-use)
- **ODP maintenance.** OA67938 / UJ97499 is the ODP 2025 Q2 maintenance (release 110). — [IBM APAR OA67938](https://www.ibm.com/support/pages/apar/OA67938)
- **Data Requester.** It is in TMS on z/OS 6.3.4 and obtained via Shopz and Fix Central. — [IBM Community, Jun 2025](https://community.ibm.com/community/user/blogs/ramanath-shanbhag/2025/06/20/introducing-ibm-z-omegamon-data-requester-bridging)
- **TMS 6.4 / Z Monitoring Suite 2.3.** It brings Web UI 2.1, Grafana 12.1.1, AI Insights 2.2 and the AI 6.1 agents. — [IBM Docs: Z Monitoring Suite 2.3](https://www.ibm.com/docs/SSRLTD_2.3.0/whats_new_v23.html)
- **IBM Z Integration for Observability** is marketed as a separate product. — [IBM product page](https://www.ibm.com/products/z-integration-observability)

### Inferences
- **Product names.** The older "OMEGAMON XE for z/OS 5.6" and "OMEGAMON for CICS 5.6" names were superseded by the "IBM Z OMEGAMON AI for …" 6.1 names. References to 5.6 are legacy, though TEMS REST PTFs may apply to TMS 6.3.x environments running 5.6 agents. This is unverified.

### Gaps
- I found no exact licensing terms: whether ODP or Integration for Observability needs separate entitlement beyond Z Monitoring Suite, or what each Suite tier includes.

## 6. Community code and examples

### Takeaway
Public community code is thin. The main item is `itm_exporter`, which uses the unsupported TEPS REST API. I found no public Python client for z/OS TEMS REST `/api/v1`. IBM's own Grafana plug-in (Data Requester) is the reference consumer of the REST API.

### Cited Findings
- **rafal-szypulka/itm_exporter** (MIT licence).
  - A Prometheus exporter for ITM, APM and OMEGAMON on port 8000 `/metrics`.
  - It uses the TEPS REST endpoint with config fields `itm_server_url` (e.g. `http://host:15200`), `datasets_uri` (`/providers/itm.TEMS/datasources/TMSAgent.%25IBM.STATIC134/datasets`), `managed_system_group`, and metrics and labels.
  - It has helper commands `listAgentTypes`, `listAttributeGroups` and `listAttributes`.
  - Its examples are distributed agents only, so z/OS support is unconfirmed.

  — [GitHub](https://github.com/rafal-szypulka/itm_exporter)
- **IBM sample Grafana dashboard.** The IBM 2023 blog mentions one built on the TEMS REST API. No link was published, and a reader's request for it went unanswered. — [IBM Community 2023](https://community.ibm.com/community/user/blogs/joseph-winterton1/2023/09/08/omegamon-rest-api-now-ready-for-your-use)
- **ITM SOAP from Perl.** IBM Support has "ITM Example Perl/SOAP" technotes, partly behind a login. — [IBM Support](https://www.ibm.com/support/pages/node/715689); [Building custom agent to run SOAP queries](https://www.ibm.com/support/pages/building-custom-agent-run-soap-queries-tems)
- **OMEGAMON and Grafana podcast.** "AIOps Corner" episode on the Grafana plug-in. — [Buzzsprout](https://www.buzzsprout.com/2370536/episodes/17429163-just-cut-and-paste-with-omegamon-and-the-new-grafana-plug-in)

### Gaps
- I did not find GitHub repositories for Python or Go clients of z/OS TEMS REST, or for ODP Kafka consumers. The search was limited, so some may exist.
- I did not review SHARE 2024–2025 presentations in detail.
