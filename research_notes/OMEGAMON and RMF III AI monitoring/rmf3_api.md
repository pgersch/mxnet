# Programmatic API access to IBM z/OS RMF Monitor III data (for an AI polling and anomaly agent)

Researched 2026-10-17. Primary source for request syntax: *z/OS V2R4 RMF Programmer's Guide*, Chapter 3 "Accessing performance data using the RMF Distributed Data Server" (IBM PDF https://www.ibm.com/docs/en/SSLTBW_2.4.0/pdf/erbb700_v2r4.pdf). I extracted its text locally and quote it below. It is a 2.4 document, but the IBM 3.2 docs still describe the same XML API (https://www.ibm.com/docs/SSLTBW_3.2.0/com.ibm.zos.v3r2.erbrp00/ddsapi.htm). Anything newer (JSON, OpenMetrics, Java DDS) is flagged as such.

## 1. DDS (GPMSERVE) HTTP API: URLs, parameters, formats, port, auth

### Takeaway
The supported programmatic interface is the RMF Distributed Data Server (GPMSERVE). It is one per sysplex and listens on HTTP port 8803 by default. Requests use the form `http(s)://host:8803/gpm/<filename>.xml?params`, and the key filenames are `rmfm3.xml` (full Monitor III report), `perform.xml` (a single metric), and `index/listmetrics/contained/root.xml` (discovery). Responses are `<ddsml>` XML. IBM's own Grafana client also requests JSON (`Accept: application/json`, with the extensionless path `/gpm/rmfm3`) from the newer DDS. Authentication is HTTP Basic against SAF (RACF), plus HTTPS through AT-TLS. Client certificates are optional.

### Cited Findings
**Architecture**
- There are two DDS components. GPMSERVE serves z/OS sysplexes, and GPM4CIM (RMF XP) serves AIX and Linux. Each listens on its own port, and both return XML with the same syntax. — [IBM Docs 3.2 ddsapi](https://www.ibm.com/docs/SSLTBW_3.2.0/com.ibm.zos.v3r2.erbrp00/ddsapi.htm)
- Placement: run one DDS per monitored sysplex, on the system with the highest RMF release. The Monitor III gatherer must run on every system with identical MINTIME and SYNC options, and USS and TCP/IP must be fully up. — [IBM Docs 3.2, Setting up the DDS](https://www.ibm.com/docs/SSLTBW_3.2.0/com.ibm.zos.v3r2.erbru00/admdds.htm)
- **New in z/OS 3.x:** the 3.2 docs show the `GPMSERVE` procedure as Java-based (`ROOT='/usr/lpp/gpm'`, `JAVAVER='21'`, with IBM Semeru v17, v21 or v25 supported). The *legacy* DDS runs from procedure `GPMSERV1`. DDS also supports HTTP compression and zEDC. — [IBM Docs 3.2 admdds](https://www.ibm.com/docs/SSLTBW_3.2.0/com.ibm.zos.v3r2.erbru00/admdds.htm)
- RMF 3.1 highlights included "DDS modernization, Grafana based User Interface, alerting and thresholding" (SHARE 2023 session abstract, no detail given). — [SHARE "RMF: The Latest and Greatest"](https://www.share.org/Events/Past-Events/Proceedings/Proceeding-Details/32031-rmf-the-latest-and-greatest)

**General syntax and request types (Programmer's Guide, Ch. 3)**
- General syntax: `http://<ddshost>:<ddsport>/gpm/<filename>?<parm_1>=<value_1>&…`. — [RMF Programmer's Guide V2R4 PDF](https://www.ibm.com/docs/en/SSLTBW_2.4.0/pdf/erbb700_v2r4.pdf)
- Filename table. The request type is set by the filename, and the document type returned is shown in brackets. — [IBM Docs, request types](https://www.ibm.com/docs/SSLTBW_2.4.0/com.ibm.zos.v2r4.erbb700/files.htm)
  - `root.xml`: root sysplex resource, no parameters `<contained-resources-list>`
  - `contained.xml?resource=`: child resources `<contained-resources-list>`
  - `index.xml`: all metrics for all resources, no parameters `<metric-list>`
  - `listmetrics.xml?resource=`: metrics, and available reports, for one resource `<metric-list>`
  - `perform.xml?resource=&id=[&range=&filter=&workscope=]`: one metric `<report>`
  - `rmfm3.xml?report=&resource=[&range=]`: a full Monitor III report `<report>`
  - `rmfpp.xml?reports=|overview=[&date=&duration=...]`: Postprocessor (long-term SMF) report(s) `<postprocessor>`
  - `details.xml`, `detailsname.xml`, `filterinst.xml`, `workscopes.xml`
- Verbatim examples from the guide — [Programmer's Guide PDF](https://www.ibm.com/docs/en/SSLTBW_2.4.0/pdf/erbb700_v2r4.pdf):
  - `http://ddshost:8803/gpm/rmfm3.xml?report=CHANNEL&resource=,SYSA,MVS_IMAGE` (required: `report`, `resource`; optional: `range`)
  - `http://ddshost:8803/gpm/perform.xml?resource=,SYSA,MVS_IMAGE&id=8D0160`
  - `http://ddshost:8803/gpm/perform.xml?resource=,MVSPLEX,SYSPLEX&id=8D1000&workscope=BATCHMED,1,P`: the **performance index** (metric 8D1000) of service class BATCHMED, period 1
  - `http://ddshost:8803/gpm/perform.xml?resource=,SYSA,MVS_IMAGE&id=8D0560&workscope=,HOTTSO,S`: % workflow by job in service class HOTTSO
  - `http://ddshost:8803/gpm/contained.xml?resource=,MVSPLEX,SYSPLEX`, `.../gpm/index.xml`, `.../gpm/root.xml`, `.../gpm/workscopes.xml?resource=,MVSPLEX,SYSPLEX`
  - Report discovery: `listmetrics.xml?resource=,MVSPLEX,SYSPLEX` (sysplex-scope reports) or `listmetrics.xml?resource=,SYSA,MVS_IMAGE` (single-system reports). Reports appear as `<metric id="report_name"><format>report</format></metric>`.
  - Metric-ID discovery: search `/gpm/index.xml`. For example, `<metric id="8D0620"><description># active users</description>`.
- Parameter semantics — [Programmer's Guide PDF](https://www.ibm.com/docs/en/SSLTBW_2.4.0/pdf/erbb700_v2r4.pdf):
  - `resource=[ulq],name,type`. Examples: `,SYSA,MVS_IMAGE`, `,MVSPLEX,SYSPLEX`, `SYSA,0F,CHANNEL_PATH`, `SYSA,,PROCESSOR`. All three parts are needed when names are not unique in the sysplex.
  - `range=YYYYMMDDhhmmss[,YYYYMMDDhhmmss]` is in the monitored system's **local time**. "If you omit the end time, the defined Monitor III gatherer interval (MINTIME) is used to determine the end time."
  - `filter=` keywords are `PAT=`, `LB=`, `UB=`, `HI=`, `LO=` and `ORD=NA|ND|VA|VD|NN`, separated by `;`. The default is `HI=20;ORD=VD`, which returns only the **top 20** list values. Override it if you need everything.
  - `workscope=[ulq],name,type`, where the type is G (global), W (workload), S (service class), P (service class period), R (report class) or J (job).
- XML response skeleton: `<ddsml>` with `<server><name>RMF-DDS-Server</name><version>ZOSV2R4</version><functionality>2381</functionality><platform>z/OS</platform></server>`, then e.g. `<report><metric id="CPC"/><resource/><time-data/><caption/><row refno="1">…</row>…<column-headers/></report>`. `<time-data>` contains `local-start/local-end/utc-start/utc-end/local-prev/local-next/display-start/display-end/gatherer-interval unit="seconds"/data-range`. Example from the guide: `<gatherer-interval unit="seconds">30</gatherer-interval>`. Errors and warnings arrive as `<message id="GPMnnnnI"><description/><severity/>`. — [Programmer's Guide PDF](https://www.ibm.com/docs/en/SSLTBW_2.4.0/pdf/erbb700_v2r4.pdf)
- Monitor III reports available as tables (the same names work as `report=` values): CACHDET, CACHSUM, CFACT, CFOVER, CFSYS, CHANNEL, CPC, CRYPTO, DELAY, DEV, DEVR, DI, DSND, DSNJ, DSNV, EADM, ENCLAVE, ENQ, ENQR, HSM, IOQUEUE, JES, JOB, LOCKSP, LOCKSU, OPD, PCIE, PROC, PROCU, RLSDS, RLSLRU, RLSSC, SPACED, SPACEG, STOR, STORC, STORCR, STORF, STORM, STORR, STORS, SYSENQ, SYSINFO, SYSRG, SYSRTD, SYSSUM, SYSWKM, USAGE, WFEX, XCF, ZFSFS, ZFSKN, ZFSOVW. — [Programmer's Guide PDF, Ch. 8 data tables](https://www.ibm.com/docs/en/SSLTBW_2.4.0/pdf/erbb700_v2r4.pdf)
- A third-party confirmation of the URL form is BMC CMF's emulation of the DDS: `http://sysA:8803/gpm/rmfm3.xml?report=CPC&resource=",sysB,MVS_IMAGE"`. — [BMC CMF docs](https://docs.bmc.com/docs/opsmcmf/63/en/requesting-cmf-report-data-through-rest-1034646533.html)

**JSON (newer DDS)**
- IBM's Grafana plugin DDS client (Go) uses the path constants `/gpm/rmfm3`, `/gpm/perform`, `/gpm/index`, `/gpm/contained`, `/gpm/root`, plus `/gpm/include/reptrans.xsl` for column-header names. It sets `Accept: application/json` and checks the response `Content-Type`. If it gets a 404 or 406, or a non-JSON response, it **falls back to appending `.xml`**. Its `DefaultMinTime = 100`, and it reads MINTIME from the response time-data. The accepted informational message IDs are GPM0501I/0502I/0504I-0507I and GPM0709I, and HTTP 401 maps to "not authorized to access DDS". — [pkg.go.dev IBM/RMF dds](https://pkg.go.dev/github.com/IBM/RMF/grafana/rmf-app/pkg/plugin/dds); [client.go source](https://raw.githubusercontent.com/IBM/RMF/main/grafana/rmf-app/pkg/plugin/dds/client.go)
- An IBM Ideas comment (Sep 2024, unofficial) says the "New DDS on z/OS 3.1" supports JSON for most data via `Accept: application/json`. — [IBM Ideas ZOS-I-1804](https://ibm-z-hardware-and-operating-systems.ideas.ibm.com/ideas/ZOS-I-1804)

**OpenMetrics / Prometheus (newest, 2025)**
- APAR **OA67701** (closed 2025-08-08; PTFs UJ97814 for R7E0 and UJ97815 for R7F0) adds OpenMetrics to GPMSERVE with a new **`/metrics/m3`** endpoint, for consumption by Prometheus, VictoriaMetrics or InfluxDB. — [IBM APAR OA67701](https://www.ibm.com/support/pages/apar/OA67701)
- Applies to RMF 3.1 and 3.2. The `groups` parameter (e.g. `?groups=cpcs,lpars`) filters output. Groups and metric names are defined in parmlib member **GPMOMC** (sample in SYS1.SERBPWSV) through a YAML file that reloads dynamically. "The polling interval must match the Monitor III MINTIME." Example: `https://dds:8803/metrics/m3`. — [IBM Community blog, Jan 2026](https://community.ibm.com/community/user/blogs/natasha-kanga/2026/01/16/openmetrics-rmf-support)
- Sample scrape config: `metrics_path: /metrics/m3`, `scrape_interval: 100s # Equal to the Monitor III mintime`, `scrape_timeout: 50s`, `params: groups: cpcs,lpars`, target `ddshostname:8803`. — [IBM RMF Grafana docs, Prometheus sample dashboards](https://ibm.github.io/RMF/grafana/rmf-app/prometheus_sample_db.html)

**Auth / transport (GPMSRV00 parmlib)**
- Default GPMSRV00 (2.4+ docs): `CACHESLOTS(4)`, `DEBUG_LEVEL(0)`, `SERVERHOST(*)`, `MAXSESSIONS_INET(5)`, `SESSION_PORT(8801)` (RMF PM), `TIMEOUT(0)`, `MAXSESSIONS_HTTP(20)`, `HTTP_PORT(8803)`, `HTTP_ALLOW(*)`, `HTTP_NOAUTH()` ("No server can access without auth."), `EXCLUDE_REPORTS()`, `HTTPS(ATTLS)` ("AT-TLS setup required"; it refuses HTTP connections that AT-TLS does not secure, while `NO` disables that check), `CLIENT_CERT(NONE)` (`ACCEPT` takes the user ID from an AT-TLS client certificate). `CACHESLOTS` range is 3–32, with one slot per MINTIME. — [IBM Docs, DDS options](https://www.ibm.com/support/knowledgecenter/SSLTBW_2.4.0/com.ibm.zos.v2r4.erbb200/ddsopti.htm)
- In practice: userid/password over HTTP Basic, validated by SAF. Access is controlled by a RACF `APPL` class profile `GPMSERVE` (`RDEFINE APPL GPMSERVE UACC(NONE)`; `PERMIT ... ACCESS(READ)`; `SETROPTS RACLIST(APPL) REFRESH`). GPMSERVE caches access after first use, so a permission change requires a server restart. The AT-TLS policy uses port 8803 with HandshakeRole Server. Operator commands: `S GPMSERVE`, `F GPMSERVE,DISPLAY`, and `F GPMSERVE,APPL=DISPLAY` on the newer version. RMF III must be running (`F RMF,START III`), or the old DDS abends S0C4. — [Colin Paice blog](https://colinpaice.blog/tag/rmf/)
- Postprocessor-via-DDS also needs a SURROGAT profile for `GPMUID.SUBMIT`, JESSPOOL and JESJOBS profiles, plus JES2. This path is not needed for Monitor III polling. — [IBM Docs 3.2 admdds](https://www.ibm.com/docs/SSLTBW_3.2.0/com.ibm.zos.v3r2.erbru00/admdds.htm)

### Inferences
- For a polling agent, the recommended pattern is:
  1. Call `root.xml` and `contained.xml` once to discover the sysplex and system names.
  2. Call `listmetrics.xml` per scope to discover the available reports.
  3. Each MINTIME, call `rmfm3.xml?report=X&resource=,SYS,MVS_IMAGE` for system-scope reports, or `resource=,PLEX,SYSPLEX` for sysplex reports such as SYSSUM, CPC and CFOVER.
  4. Use `perform.xml` with metric IDs for cheap single-value checks such as the PI of critical service class periods.
  If OA67701 is applied, scraping `/metrics/m3` into Prometheus and having the agent query PromQL is lower-effort and yields time series natively.
- Prefer JSON via `Accept: application/json` on 3.1+ with the XML fallback, mirroring IBM's client logic.
- Use a dedicated RACF ID with READ on APPL GPMSERVE. Keep `HTTPS(ATTLS)`, and don't use `HTTP_NOAUTH` for the agent host.
- Sample code, illustrative and written for these notes (not from a source):
  ```bash
  curl -s -u AGENTID:$PW -H 'Accept: application/json' \
    'https://dds.example:8803/gpm/rmfm3?report=SYSINFO&resource=,SYSA,MVS_IMAGE'
  curl -s -u AGENTID:$PW 'https://dds.example:8803/gpm/rmfm3.xml?report=WFEX&resource=,SYSA,MVS_IMAGE'
  ```
  ```python
  import requests, xml.etree.ElementTree as ET
  r = requests.get(f"{base}/gpm/rmfm3.xml", params={"report":"CPC","resource":",SYSA,MVS_IMAGE"},
                   auth=(user, pw), timeout=60, verify=ca_bundle)
  root = ET.fromstring(r.content)
  for msg in root.iter("message"): print(msg.get("id"), msg.findtext("description"))
  rep = root.find("report"); td = rep.find("time-data")
  interval = int(td.findtext("gatherer-interval"))         # MINTIME seconds
  hdrs = [c.text for c in rep.find("column-headers")]     # column ids
  rows = [[c.text for c in row] for row in rep.findall("row")]
  ```
  Verify the child tag names of `<row>` and `<column-headers>` (`<col>`) against a live response. The guide's example elides them.

### Gaps
- I could not retrieve the official z/OS 3.1/3.2 page documenting the JSON format or schema. JSON evidence comes from IBM's own client code and an unofficial Ideas comment.
- I could not retrieve the IBM "OpenMetrics Endpoints" doc page. I have no confirmed list of `groups` values beyond `cpcs,lpars`, no metric name list, and no statement of whether `/metrics/m3` requires Basic auth. It presumably honors the same HTTP_NOAUTH/SAF rules, but that is unverified.
- The exact `<row>`/`<col>` XML child structure and the `/gpm/include/reptrans.xsl` header mapping are not shown in full in the guide text I extracted.
- I found no source for an official IBM Python SDK. No IBM Python examples were found.

## 2. z/OSMF, Zowe, the Grafana plugin, Prometheus, and IBM analytics integrations

### Takeaway
z/OSMF Resource Monitoring is a UI that consumes the DDS. I found no documented z/OSMF REST API for RMF data, and an IBM Idea requesting one is still "Future consideration". IBM's own programmatic integrations are the **IBM RMF for z/OS Grafana plugin** (`ibm-rmf-app`, open source at github.com/IBM/RMF) and the DDS **OpenMetrics** endpoint. **ZEBRA** (Zowe incubator) is an older open-source Node.js DDS-to-JSON/Prometheus bridge.

### Cited Findings
- z/OSMF Resource Monitoring takes its z/OS input from GPMSERVE, and each sysplex needs an active DDS. — [IBM Docs, z/OSMF Resource Monitoring](https://www.ibm.com/support/knowledgecenter/SSLTBW_2.2.0/com.ibm.zos.v2r2.erbb200/zosmf.htm)
- An IBM Idea requests "RESTful APIs for z/OSMF supporting RMF monitoring services" and has status Future consideration. — [IBM Ideas ZOS-I-1785](https://ibm-z-hardware-and-operating-systems.ideas.ibm.com/ideas/ZOS-I-1785)
- Grafana plugin `ibm-rmf-app` covers Monitor III metrics and reports, near-real-time and historical. Latest listed version is 2.0.2, which requires Grafana ≥11. Version 2.0.0 added import of RMF PM dashboards. — [Grafana plugin page](https://grafana.com/grafana/plugins/ibm-rmf-app/); [changelog](https://grafana.com/grafana/plugins/ibm-rmf-app/changelog/)
- Plugin docs cover data sources, alerts, variables, historical data, OMEGAMON Web UI integration, and launching Grafana from z/OSMF Resource Monitoring. The repo is https://github.com/IBM/RMF. — [ibm.github.io/RMF](https://ibm.github.io/RMF/grafana/rmf-app/)
- Data source config: URL `http(s)://host:port[/path]` (path only behind a reverse proxy), Basic Auth with a DDS user and password, a 60s default timeout, compression on (requires DDS maintenance OA67541), "Skip TLS verify" off by default, and a cache defaulting to 1024 MB. — [Creating RMF data sources](https://ibm.github.io/RMF/grafana/rmf-app/create_datasources.html)
- The RMF 3.1 Grafana plugin shipped sample dashboards for Performance Index by service class period and report class period. RMF 3.1 also enhanced Monitor III reports: STORC (RUCSA/ERUCSA), CPC (2-decimal LPAR usage), WLMGL (enclave columns), and PROCU (ms). — [IBM Community blog Dec 2023](https://community.ibm.com/community/user/blogs/ramanath-shanbhag/2023/12/06/latest-modernizations-in-resource-measurement-faci)
- OpenMetrics `/metrics/m3` (OA67701) works with Prometheus, Telegraf and VictoriaMetrics, and the Prometheus sample dashboards ship with the plugin. — [APAR OA67701](https://www.ibm.com/support/pages/apar/OA67701); [IBM blog Jan 2026](https://community.ibm.com/community/user/blogs/natasha-kanga/2026/01/16/openmetrics-rmf-support)
- ZEBRA (Zowe incubator, Node.js) wraps the DDS. It exposes `GET /v1/{lpar}/rmfm3/{report}`, `/v1/{lpar}/rmfpp/{report}` and `/v1/{lpar}/rmf?id=`, plus `/prommetric`, and returns JSON (`data`, `msg`, `err`). Its configurable `rmf3filename` defaults to `rmfm3.xml`. Supported Monitor III reports: CHANNEL, CPC, DELAY, DEV, DEVR, DSND, EADM, ENCLAVE, ENQ, HSM, JES, OPD, PROC, PROCU, STOR, STORC, STORCR, SYSINFO, SYSSUM, USAGE. — [github.com/zowe/zebra](https://github.com/zowe/zebra); [OMP blog](https://openmainframeproject.org/blog/zebra-get-to-know-rmf-using-modern-json-format/)
- IBM Z Common Data Provider has an "RMF III report data" collector, so CDP can pull Monitor III reports (via DDS) and stream them. — [IBM Z CDP 5.1 docs](https://www.ibm.com/docs/en/zcdp/5.1.0?topic=collector-rmf-iii-report-data)
- OMEGAMON maintenance notes say its RMF DDS requests were changed to use `/gpm/rmfm3.xml`. — [APAR OA67597](https://www.ibm.com/support/pages/apar/OA67597) (per search snippet; not opened)

### Inferences
- ZEBRA's documentation requires Node 8, so the project looks stale. Treat it as reference code for parsing DDS XML, not as a production dependency.
- For a feed into IBM Z Anomaly Analytics or IZPCA, the likely path is CDP's RMF III collector. I did not verify the details.

### Gaps
- I found no specific documentation on IBM Z Anomaly Analytics or IBM Z Performance & Capacity Analytics consuming Monitor III directly.
- I found no Zowe CLI plugin for RMF.

## 3. Key Monitor III reports and problem thresholds

### Takeaway
For automated detection, the highest-value reports are:
- **SYSINFO** for workflow %, delay counts by resource, and CPU % per group/class
- **WFEX**, which lists exceptions already evaluated by RMF (Name, Reason, Delay/critical value, Possible cause)
- **SYSSUM/SYSRTD** for goal attainment and PI. The PI metric is ID `8D1000` with a service-class-period workscope.
- **CPC** for LPAR and physical utilization
- **DELAY/PROC/DEV/STOR(F)** for drill-down
- **ENCLAVE, CFOVER, CACHSUM, XCF**

The canonical threshold is PI > 1.0, meaning the goal was missed, with PI > 2 often flagged as severe. Other numeric thresholds are site-defined in WFEX.

### Cited Findings
- SYSINFO (ERBSYST3) columns include:
  - workflow % (SYSWFLVC)
  - active and total users
  - transactions/s
  - response time in ms (SYSRSPM)
  - average number using PROC and DEV
  - average number delayed for PROC, DEV, STOR, SUBS, OPER, ENQ, JES, HSM, XCF, Mount and Message
  - % of GP capacity: SYSCPUVC, SYSSRBVC, SYSTCBVC, SYSEAPVC
  - zAAP % (SYSIFAVC)

  — [Programmer's Guide PDF](https://www.ibm.com/docs/en/SSLTBW_2.4.0/pdf/erbb700_v2r4.pdf)
- WFEX (ERBWFXT3) columns: WFXNAME (Name), WFXREASN (Reason), WFXDELAY (Delay / "Critical val."), WFXPCAUS (Possible cause), WFXATTR. — [Programmer's Guide PDF](https://www.ibm.com/docs/en/SSLTBW_2.4.0/pdf/erbb700_v2r4.pdf)
- WFEX lets you define thresholds for CPU utilization, response times, user counts, storage and more, and Monitor III flags them when reached. IBM's example sets a critical TCB+SRB threshold of 90% and shows how to alert at 80% instead (procedure ERBR3SYS `tcbsrb_limit`, run with RMFM3B `report=sysinfo`). — [IBM Docs, exception handling](https://www.ibm.com/support/knowledgecenter/SSLTBW_2.2.0/com.ibm.zos.v2r2.erbb200/erbb200327.htm) (via search snippet)
- IBM's sample WFEX REXX handler triggers on `*PROC` with reason `CPUS%` ≥ 90 and on `*ECSA*` with reason `SECS%` ≥ 60. These are sample values, not defaults. — [IBM Docs r3wfx sample](https://www.ibm.com/support/knowledgecenter/SSLTBW_2.2.0/com.ibm.zos.v2r2.erbb200/r3wfx.htm)
- PI = 1.0 means the goal is exactly met, and PI > 1.0 means it was missed. In WLMGL reporting, yellow marks a missed goal and red marks PI > 2. — [SHARE handout "WLM – Are you Set Up to Fail"](https://share.confex.com/share/123/webprogram/Handout/Session15707/WLM%20-%20Are%20you%20Set%20Up%20to%20Fail.pdf); [IntelliMagic](https://www.intellimagic.com/?p=9918)
- Some tools treat 0.8 ≤ PI ≤ 1.2 as met, so a strict >1 rule will raise noise. — [SHARE blog, WLM Policy Advisor](https://blog.share.org/Article/the-wlm-policy-advisor-policies-made-simple)
- PI only says that a goal was missed. To diagnose why, look at using versus delay components (CPU contention versus DASD). — [IntelliMagic](https://www.intellimagic.com/?p=9918)
- Metric `8D1000` is the performance index, and `8D0560` is % workflow by job. — [Programmer's Guide PDF](https://www.ibm.com/docs/en/SSLTBW_2.4.0/pdf/erbb700_v2r4.pdf)

### Inferences
These are heuristics for agent rules. They are my synthesis, not IBM defaults.
- Alert when PI > 1 for an importance 1–2 period across ≥2–3 consecutive MINTIME intervals, and page when PI > 2.
- Treat any WFEX row as a candidate event, using `Possible cause` as the explanation text.
- In SYSINFO/DELAY, a falling workflow % together with a dominant delay type (PROC, DEV, STOR or ENQ) localizes the bottleneck.
- Track CPC LPAR/physical busy near 90–100%, the `CPUS% ≥ 90` pattern from the IBM sample, and ECSA/CSA usage above 60–80%.
- Baseline-relative anomaly detection (time-of-day seasonality) suits the agent better than fixed thresholds, because WFEX thresholds are already site-tuned.

### Gaps
- I found no authoritative IBM list of default WFEX thresholds or of paging-rate, device-response or CF thresholds. Report Analysis manuals were not retrieved.
- I did not retrieve full column lists for the CPC, DELAY, DEV, STORF, ENCLAVE, CFOVER and CACHSUM tables, though they exist in Programmer's Guide Ch. 8.

## 4. MINTIME and polling frequency

### Takeaway
Monitor III builds one sample set per MINTIME, 100 seconds by default, synchronized to the hour with SYNC(0M). The DDS, the Grafana client and the OpenMetrics scrape guidance all align to MINTIME. Polling faster than MINTIME returns the same interval again.

### Cited Findings
- Default gatherer options are MINTIME(100), which builds a set of samples every 100 s, and SYNC(0M), which synchronizes MINTIME on the hour. — [IBM Docs, default gatherer options](https://www.ibm.com/support/knowledgecenter/SSLTBW_2.2.0/com.ibm.zos.v2r2.erbb200/deflt3.htm)
- SYNC must be specified for sysplex reports, and MINTIME and SYNC must be identical on all systems for the DDS. — [IBM SYNC doc](https://www.ibm.com/support/knowledgecenter/SSLTBW_2.2.0/com.ibm.zos.v2r2.erbb200/syn.htm); [IBM Docs 3.2 admdds](https://www.ibm.com/docs/SSLTBW_3.2.0/com.ibm.zos.v3r2.erbru00/admdds.htm)
- A `range` with no end time defaults to MINTIME. `<time-data>` returns `gatherer-interval` and `local-prev`/`local-next`, which let the client walk intervals. — [Programmer's Guide PDF](https://www.ibm.com/docs/en/SSLTBW_2.4.0/pdf/erbb700_v2r4.pdf)
- DDS `CACHESLOTS(4)` sets cache entries per MINTIME (range 3–32). — [DDS options](https://www.ibm.com/support/knowledgecenter/SSLTBW_2.4.0/com.ibm.zos.v2r4.erbb200/ddsopti.htm)
- Prometheus guidance says scrape_interval should equal MINTIME (100s), and the blog says the polling interval must match MINTIME. — [ibm.github.io Prometheus](https://ibm.github.io/RMF/grafana/rmf-app/prometheus_sample_db.html); [IBM blog](https://community.ibm.com/community/user/blogs/natasha-kanga/2026/01/16/openmetrics-rmf-support)
- IBM's Grafana client uses DefaultMinTime=100, reads MINTIME from the response, and refreshes cached headers every 15 min. — [pkg.go.dev](https://pkg.go.dev/github.com/IBM/RMF/grafana/rmf-app/pkg/plugin/dds)

### Inferences
- The agent should read `gatherer-interval` at startup and then poll once per MINTIME, aligned shortly after the boundary (e.g. MINTIME + 5–10 s). It should dedupe on `local-end` and backfill missed intervals using `range=` and `local-prev`.
- The default `MAXSESSIONS_HTTP(20)` caps concurrency. Polling about 10 reports × N systems in parallel can hit that cap.
- For lookback (e.g. 1 h) beyond the DDS cache, `range` queries read the Monitor III VSAM data sets, but only if the gatherer writes them (WSTOR/DATASET options; not verified).

### Gaps
- The valid MINTIME range was not confirmed. The IBM min.htm page was not retrieved.
- I found no IBM guidance on load or CPU cost of frequent DDS polling.
