# Programmatic Email and Microsoft Teams Alerting from a Monitoring Program / AI Agent (state as of Oct 2026)

## Teams: Connectors/Incoming Webhook retirement, Workflows replacement, Graph chat API, bots, cards, limits

### Takeaway
Office 365 (Microsoft 365) Connectors, including the classic "Incoming Webhook" (`*.webhook.office.com` URLs), are **retired**. The final shutdown rolled out **May 18–22, 2026**, and those URLs no longer work. Today the simplest way for a monitor to push an alert into a **channel or group chat** is a **Teams Workflows (Power Automate) webhook**. You create it from a template such as "Send webhook alerts to a channel" or "...to a chat", or from scratch with the trigger "When a Teams webhook request is received", then POST an Adaptive Card JSON to the generated URL. Microsoft Graph `POST /chats/{id}/messages` works for normal sending only with **delegated** permission (`ChatMessage.Send`): a signed-in user identity. Its only **application** permission is `Teamwork.Migrate.All`, which is for importing messages into chats in migration mode, so a daemon cannot use it for live alerts. The robust app-identity alternative is a **Teams bot/agent** (Teams SDK, formerly Teams AI Library / Bot Framework) installed in the group chat or team, which sends proactive messages to a stored `conversationId`.

### Cited Findings
**Connector / Incoming Webhook retirement (RETIRED)**
- Retirement history: the original plan blocked new connectors on Aug 15, 2024 and stopped all connectors on Oct 1, 2024. That was extended through Dec 2025 and then to Mar 31, 2026. On Feb 5, 2026, MessageCard support arrived in Workflows and the deadline moved to Apr 30, 2026. In the final update (Apr 14, 2026), private channel selection in Workflows was fully rolled out by Apr 20, 2026, and the retirement **rollout began May 18, 2026 and completed May 22, 2026**. After that, "Office 365 Connectors will no longer function." — [Microsoft 365 Dev Blog: Retirement of Office 365 connectors within Microsoft Teams](https://devblogs.microsoft.com/microsoft365dev/retirement-of-office-365-connectors-within-microsoft-teams/)
- Third-party summaries agree that Incoming Webhooks were switched off May 18–22, 2026. Older articles that cite Dec 2025 or Mar 2026 are outdated. — [webhookrelay.com 2026 guide](https://webhookrelay.com/blog/microsoft-teams-webhook/); [tiagoscarvalho.com](https://www.tiagoscarvalho.com/microsoft-teams/office-365-connectors-retirement-teams); [MC1181996](https://msmessagecenter.com/MC1181996)
- Workflows limitations (Apr 2026): the bot icon and display name can't be customized (posts appear as the Workflows/Flow bot), and MessageCard payloads render, but their HttpPost action buttons do not. — [M365 Dev Blog](https://devblogs.microsoft.com/microsoft365dev/retirement-of-office-365-connectors-within-microsoft-teams/)
- The Apprise library's legacy `msteams://` (Incoming Webhook) service page says: "This service has been retired upstream." — [Apprise msteams docs](https://appriseit.com/services/msteams/)

**Workflows (Power Automate) webhook: current replacement**
- From the Microsoft Learn page (updated Aug 2026): "To automatically post to a chat or channel when a webhook request is received, use the predefined workflow templates or create a workflow from scratch using the **When a Teams webhook request is received** trigger." Setup: open the channel's "..." menu, choose **Workflows**, pick a template such as **Send webhook alerts to a channel**, configure it, save, and **copy the webhook link**. — [Learn: Create an Incoming Webhook (Workflows section)](https://learn.microsoft.com/en-us/microsoftteams/platform/webhooks-and-connectors/how-to/add-incoming-webhook)
- Known issues and limitations on that page: "Workflows support both Adaptive Cards and Message Card format (button rendering won't be supported)." Also: "Workflows are linked only to specific users (owners of the workflow) and not to a Teams team or channel. Workflows can become orphan flows in the absence of an owner if no co-owners assigned." Admins can add co-owners. Same page: "Workflows app can post as a flow bot in shared channels. Support for posting as a flow bot in private channels is under development." This conflicts with the dev blog's Apr 2026 statement that private channel selection had rolled out, so check your tenant. — [Learn](https://learn.microsoft.com/en-us/microsoftteams/platform/webhooks-and-connectors/how-to/add-incoming-webhook)
- Limits: "The message size limit is 28 KB." "If more than four requests are made in a second, the client connection is throttled". Microsoft recommends retry with exponential backoff and handling HTTP 429. — [Learn](https://learn.microsoft.com/en-us/microsoftteams/platform/webhooks-and-connectors/how-to/add-incoming-webhook)
- Adaptive Card payload shape (official sample): `{"type":"message","attachments":[{"contentType":"application/vnd.microsoft.card.adaptive","content":{"$schema":"http://adaptivecards.io/schemas/adaptive-card.json","type":"AdaptiveCard","version":"1.2","body":[{"type":"TextBlock","text":"..."}]}}]}` — [Learn](https://learn.microsoft.com/en-us/microsoftteams/platform/webhooks-and-connectors/how-to/add-incoming-webhook)
- Pitfall: if the posted JSON lacks an `attachments` array, the default template fails with `ExpressionEvaluationFailed ... 'foreach' expression '@triggerOutputs()?['body']?['attachments']' is of type 'Null'`. — [Power Automate community thread](https://jgvjg48436.lithium.com/t5/Using-Flows/Using-Post-to-a-channel-when-a-webhook-request-is-received/m-p/2839048); [Cribl knowledge article](https://knowledge.cribl.io/microsoft-47/microsoft-teams-notifications-fail-after-migrating-to-workflows-2398)
- Some vendor guides configure the flow trigger's "Who can trigger the flow?" as **Anyone**. In that setup the URL (which carries a signature) is effectively the secret. — [Check Point Playblocks guide](https://sc1.checkpoint.com/documents/Infinity_Portal/WebAdminGuides/EN/Playblocks-Admin-Guide/Topics/appendixc-creatingworkflowteams/appendix-c-creating-workflow-for-microsoft-teams-notification.html)

**Microsoft Graph: send to a group chat**
- `POST /chats/{chat-id}/messages` permissions: Delegated (work/school) **ChatMessage.Send** (least privileged), or the higher-privileged Chat.ReadWrite or Group.ReadWrite.All. Personal accounts are not supported. **Application: Teamwork.Migrate.All** only. — [Learn: Send message in a chat](https://learn.microsoft.com/en-us/graph/api/chat-post-messages?view=graph-rest-1.0)
- From the same page: "The permission scope `Teamwork.Migrate.All` is required for this scenario. The target chat must be in migration mode" (the import-messages example). Also: "It's a violation of the terms of use to use Microsoft Teams as a log file. Only send messages that people will read." And: "This API can't create a new chat; you must use the list chats method to retrieve the ID of an existing chat." Chat IDs look like `19:...@thread.v2`. — [Learn](https://learn.microsoft.com/en-us/graph/api/chat-post-messages?view=graph-rest-1.0)
- Minimal body: `{"body":{"content":"Hello world"}}`. Returns `201 Created`. — [Learn](https://learn.microsoft.com/en-us/graph/api/chat-post-messages?view=graph-rest-1.0)

**Bots/agents: proactive messages to a group chat or channel (app identity)**
- "To send a proactive message to a user, a group chat, or a team ... For a group chat or team, the app that contains your agent must be first installed in that location." "You can't create a new group chat or a new channel in a team with proactive messaging." The app stores the `conversationId` (received in install/conversationUpdate events) and sends with `app.send(conversation_id, activity)` in the Teams SDK (Python, TypeScript, C#). — [Learn: Send proactive messages (Teams SDK), updated Oct 2026](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/conversations/send-proactive-messages)
- Global service URLs to use when no incoming activity is available: Public `https://smba.trafficmanager.net/teams/`, plus GCC, GCC High and DoD variants. Microsoft warns against hardcoding them. — [Learn](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/conversations/send-proactive-messages)
- A blocked or uninstalled bot gets `403` with `subCode: MessageWritesBlocked`. Bots can update or delete their earlier messages via `PUT {serviceUrl}/v3/conversations/{conversationId}/activities/{activityId}`, which is useful for "alert resolved" edits. — [Learn](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/conversations/send-proactive-messages)
- Microsoft suggests a "notification bot" built with the **Microsoft 365 Agents Toolkit** (formerly Teams Toolkit) as a more capable alternative to an incoming webhook. — [Learn add-incoming-webhook](https://learn.microsoft.com/en-us/microsoftteams/platform/webhooks-and-connectors/how-to/add-incoming-webhook)

### Inferences
- Recommended order for an unattended mainframe monitor:
  1. **Workflows webhook** to a channel or group chat. Zero app registration; you POST an Adaptive Card.
  2. A **Teams bot** installed in the group chat or channel. It has an app identity, supports updatable and threaded messages, and needs no user license token.
  3. **Graph delegated** `ChatMessage.Send` with a dedicated service *user* account and a refresh token. This works, but it is brittle (MFA/Conditional Access, token expiry) and ties posts to a person.
  
  Application-only Graph posting to chats is **not** viable, because Teamwork.Migrate.All is for migration only.
- A minimal Python sender for a Workflows webhook (my own code, built on the documented payload):
  ```python
  import requests, time
  def post_teams(url, title, severity, text, runbook):
      card = {"type":"message","attachments":[{"contentType":"application/vnd.microsoft.card.adaptive",
        "content":{"$schema":"http://adaptivecards.io/schemas/adaptive-card.json","type":"AdaptiveCard","version":"1.4",
          "body":[{"type":"TextBlock","size":"Large","weight":"Bolder","text":f"[{severity}] {title}",
                   "color":"Attention" if severity in ("CRITICAL","HIGH") else "Warning","wrap":True},
                  {"type":"TextBlock","text":text,"wrap":True}],
          "actions":[{"type":"Action.OpenUrl","title":"Runbook","url":runbook}]}}]}
      for attempt in range(5):
          r = requests.post(url, json=card, timeout=15)
          if r.status_code < 300: return
          if r.status_code == 429 or r.status_code >= 500:
              time.sleep(int(r.headers.get("Retry-After", 2**attempt))); continue
          r.raise_for_status()
  ```
- Graph delegated chat post (service account, MSAL device-code or refresh token): `POST https://graph.microsoft.com/v1.0/chats/{chat-id}/messages` with `{"body":{"contentType":"html","content":"<b>CRITICAL</b> ..."}}`. To find the chat ID, call `GET /me/chats?$filter=chatType eq 'group'` and match on `topic`, or copy it from the Teams web client URL.
- Treat the Workflows URL as a secret: store it in a vault and rotate it by recreating the flow. Assign co-owners to the flow so the alert path doesn't die when its creator leaves.

### Gaps
- I did not find an official Microsoft Learn page that documents the exact template name "Send webhook alerts to a chat" or its expected schema. The Learn page confirms only that Workflows can post "to a channel or chat."
- I did not verify Microsoft Graph Teams-specific throttling numbers (per-app/per-tenant messages per second for chat/channel posts) or bot proactive messaging rate limits (the "rate-limit" page was not fetched).
- I did not verify the current host and format of Workflows trigger URLs. The reported migration from `*.logic.azure.com` to `*.powerplatform.com`/`environment.api.powerplatform.com` URLs in 2025 was not checked against a source.
- Whether Graph `POST /teams/{id}/channels/{id}/messages` has any non-migration application permission was not re-verified. Historically it is also Teamwork.Migrate.All only for app-only access.
- Private channel support for Workflows conflicts between the dev blog (rolled out Apr 2026) and the Learn page ("under development").

## Email: SMTP AUTH (basic vs OAuth), Graph sendMail, HVE, relay, z/OS

### Takeaway
For Exchange Online, **Basic auth for SMTP AUTH client submission** is still working, but it will be **disabled by default for existing tenants at the end of December 2026**. Admins can re-enable it for now, and Microsoft will announce a final removal date in H2 2027. New tenants created after Dec 2026 won't have it. Use one of these instead:
- **Microsoft Graph `sendMail`** with the application permission `Mail.Send`, scoped to a single alert mailbox via **Exchange RBAC for Applications**. RBAC for Applications replaces Application Access Policies.
- **SMTP AUTH with OAuth2 XOAUTH2** (client-credentials with `SMTP.SendAsApp` plus an Exchange service principal).
- An **IP-based connector relay**, which is unaffected by the change.
- **High Volume Email (HVE)**, GA since Mar 31, 2026. It is internal recipients only, billed from Jun 1, 2026, and keeps Basic auth until Sept 2028.

On z/OS, **CSSMTP** is the only mail client: SMTPD and sendmail were removed in V2R3. CSSMTP supports AT-TLS but does **not** do SMTP AUTH, so it must target a relay or connector that accepts it by IP or certificate.

### Cited Findings
**SMTP AUTH basic-auth timeline (DEPRECATED, phased)**
- In Microsoft's Jan 27, 2026 update, Basic auth behavior stays unchanged until **end of December 2026**. At that point it is **disabled by default for existing tenants**, and admins can re-enable it. Tenants created after Dec 2026 have Basic auth unavailable by default, and OAuth is the supported method. — [Exchange Team: Updated SMTP AUTH Basic Auth Deprecation Timeline](https://techcommunity.microsoft.com/blog/exchange/updated-exchange-online-smtp-auth-basic-authentication-deprecation-timeline/4489835); [MC786329](https://mc.merill.net/message/MC786329). I could not fetch the article body directly, so these details come from search summaries of these pages.
- The final removal date will be announced in H2 2027. — [office365itpros, Jan 29 2026](https://office365itpros.com/2026/01/29/smtp-auth-basic-retirement/)
- The earlier dates (Mar 1 and Apr 30, 2026) are superseded. — [Okta KB (outdated dates)](https://support.okta.com/help/s/article/microsoft-smtp-authentication-end-of-support-march-1st-2026?language=en_US); [original 2024 announcement](https://techcommunity.microsoft.com/blog/exchange/exchange-online-to-retire-basic-auth-for-client-submission-smtp-auth/4114750)
- The change affects only Client Submission (smtp.office365.com:587 AUTH). IP-based relay via connectors is not affected. — [captaindns.com](https://www.captaindns.com/en/blog/microsoft-exchange-basic-auth-smtp-end)

**SMTP AUTH with OAuth2 (supported)**
- Delegated scope: `https://outlook.office.com/SMTP.Send`. App-only setup:
  1. Add the **Office 365 Exchange Online > Application permission `SMTP.SendAsApp`** and grant admin consent.
  2. Register the service principal in Exchange with `New-ServicePrincipal -AppId <APPLICATION_ID> -ObjectId <OBJECT_ID>`. Use the Enterprise Application object ID, *not* the App Registration object ID.
  3. Grant mailbox access with `Add-MailboxPermission -Identity <mailbox> -User <SP_ID> -AccessRights FullAccess`.
  4. Request tokens with scope `https://outlook.office365.com/.default`.
  
  If the app sends as another address, it also needs `Add-RecipientPermission` (SendAs). — [Learn: Authenticate IMAP/POP/SMTP using OAuth](https://learn.microsoft.com/en-us/exchange/client-developer/legacy-protocols/how-to-authenticate-an-imap-pop-smtp-application-by-using-oauth)
- SASL XOAUTH2 string: `base64("user=" + userName + "^Aauth=Bearer " + accessToken + "^A^A")`, where `^A` is `\x01`. The client sends `AUTH XOAUTH2 <b64>`, success returns `235 2.7.0`, and failure returns `535 5.7.3`. — [Learn](https://learn.microsoft.com/en-us/exchange/client-developer/legacy-protocols/how-to-authenticate-an-imap-pop-smtp-application-by-using-oauth)
- Microsoft also documents "Configure SMTP onboarding to App RBAC" (an RBAC-based alternative to mailbox FullAccess for SMTP). — [Learn](https://learn.microsoft.com/en-us/exchange/client-developer/legacy-protocols/smtp-app-rbac-onboarding)

**Microsoft Graph sendMail (recommended for cloud-connected agents)**
- Endpoints: `POST /users/{id|UPN}/sendMail` (or `/me/sendMail`). Permissions: Delegated Mail.Send, or **Application Mail.Send**. The body is `{"message":{subject, body{contentType,content}, toRecipients[...] , internetMessageHeaders[...]}, "saveToSentItems": false}`. It returns **202 Accepted**, which "doesn't indicate that the request processing has completed. Delivery ... is subject to Exchange Online limitations and throttling." Attachments are supported (fileAttachment, base64), as is MIME (base64, `Content-Type: text/plain`). — [Learn: user: sendMail](https://learn.microsoft.com/en-us/graph/api/user-sendmail?view=graph-rest-1.0)
- Application Mail.Send can by default send as **any** user. Microsoft's **RBAC for Applications in Exchange Online** "replaces Application Access Policies" and lets you grant the permission with a scope (for example, a management scope containing only the alert mailbox). "Nested group members are considered out of scope." — [Learn: RBAC for Applications](https://learn.microsoft.com/en-us/exchange/permissions-exo/application-rbac)
- A July 2026 practitioner blog reports that the New-ApplicationAccessPolicy docs now say new policies should not be created, and that RBAC for Applications is PowerShell-only with no portal UI. — [mikecrowley.us, Jul 2026](https://mikecrowley.us/2026/07/10/exchange-app-access-policy-rbac-migration/); see also [Mindcore, Feb 2026](https://blog.mindcore.dk/2026/02/microsoft-graph-remembered-to-restict-mail-send-application-permission-app-access-policies/)

**High Volume Email (HVE)**
- HVE reached GA on Mar 31, 2026, for **internal recipients** only. External sending was removed in 2025 and moved to Azure Communication Services Email. Billing started **Jun 1, 2026, at $42 per million recipients**, and admins must configure billing. — [Exchange Team: HVE GA](https://techcommunity.microsoft.com/blog/exchange/high-volume-email-reaches-general-availability-in-exchange-online/4507353); [MC1243552](https://mc.merill.net/message/MC1243552); [Help Net Security](https://www.helpnetsecurity.com/2026/04/02/exchange-online-high-volume-email-hve/)
- HVE keeps Basic authentication (username/password) until **September 2028**. — [Exchange Team: HVE continued support for Basic Authentication](https://techcommunity.microsoft.com/blog/exchange/high-volume-email-continued-support-for-basic-authentication--other-important-up/4411197)

**z/OS side**
- SMTPD and sendmail were removed in z/OS V2R3 (GA Sept 2017), which leaves **CSSMTP** as the only z/OS mail client. CSSMTP supports IPv6 and **AT-TLS**. — [SHARE session 17293](https://share.confex.com/share/125/webprogram/Session17293.html); [SHARE handout](https://share.confex.com/share/125/webprogram/Handout/Session17293/Mail%20Strategy.pdf)
- CSSMTP does not support SMTP AUTH. An IBM Ideas request says AUTH support was not delivered (RFE 28169: SASL PLAIN/Kerberos), and a practitioner blog confirms CSSMTP "does not seem to support the passing of authentication information." APAR PH66548 fixed CSSMTP wrongly attempting a TLS handshake when the server offered AUTH PLAIN/LOGIN with Secure No. — [IBM Ideas ZOS-I-218](https://ibm-z-hardware-and-operating-systems.ideas.ibm.com/ideas/ZOS-I-218); [colinpaice.blog](https://colinpaice.blog/category/email/); [APAR PH66548](https://www.ibm.com/support/pages/apar/PH66548)

### Inferences
- Because CSSMTP cannot authenticate, a z/OS-originated alert must go to either:
  - an on-prem relay (Postfix, or an Exchange receive connector) that accepts it by source IP and then forwards it to Exchange Online, or
  - an Exchange Online **inbound connector** (IP or certificate based) using the tenant's MX endpoint, with AT-TLS enabled.
  
  Alternatively, the AI agent running off-host (Linux/zCX/USS Python) can call Graph sendMail directly.
- Python example: SMTP OAuth2 with client credentials (my own code, based on the documented XOAUTH2 format):
  ```python
  import msal, smtplib, base64
  from email.message import EmailMessage
  app = msal.ConfidentialClientApplication(CLIENT_ID, authority=f"https://login.microsoftonline.com/{TENANT}", client_credential=SECRET)
  tok = app.acquire_token_for_client(scopes=["https://outlook.office365.com/.default"])["access_token"]
  msg = EmailMessage(); msg["From"]=SENDER; msg["To"]="ops@contoso.com"; msg["Subject"]="[CRITICAL] CICSPRD1 MAXTASK"; msg.set_content("...runbook: https://...")
  with smtplib.SMTP("smtp.office365.com", 587) as s:
      s.starttls(); s.ehlo()
      s.docmd("AUTH", "XOAUTH2 " + base64.b64encode(f"user={SENDER}\x01auth=Bearer {tok}\x01\x01".encode()).decode())
      s.send_message(msg)
  ```
- Python example: Graph sendMail, app-only:
  ```python
  tok = app.acquire_token_for_client(scopes=["https://graph.microsoft.com/.default"])["access_token"]
  requests.post(f"https://graph.microsoft.com/v1.0/users/{SENDER}/sendMail",
      headers={"Authorization": f"Bearer {tok}"},
      json={"message":{"subject":"[CRITICAL] ...","body":{"contentType":"HTML","content":"..."},
            "toRecipients":[{"emailAddress":{"address":"ops@contoso.com"}}],
            "internetMessageHeaders":[{"name":"X-Alert-Key","value":"SYSA:CICSPRD1:MAXTASK"}]},
            "saveToSentItems": False}, timeout=15).raise_for_status()
  ```
  Treat 202 as "queued", not "delivered".
- Prefer certificate credentials over client secrets for the Entra app, and scope Mail.Send with RBAC for Applications to one shared "alerts@" mailbox.

### Gaps
- I could not fetch the full text of the Jan 2026 Exchange Team timeline article, so the details come from search summaries and secondary sources.
- I did not verify Exchange Online sending limits (recipient rate, 10,000 recipients/day) or the Graph mail throttling numbers.
- I did not verify current IBM documentation on CSSMTP in z/OS 3.1/3.2, including whether any AUTH support has been added.
- Azure Communication Services Email (for external recipients) was not researched in detail.

## Ready-made MCP servers / libraries, and alert hygiene

### Takeaway
Microsoft's official Teams MCP server is the **Work IQ / Agent 365 Teams MCP server** (`mcp_TeamsServer`). It is in **preview**, requires Microsoft 365 Copilot licensing and tenant admin consent, posts **plain text only**, and runs as the user (Graph delegated semantics). Microsoft itself now says the app-specific M365 MCP tools are for backward compatibility and points new projects to Work IQ MCP. So for an unattended alerting agent, a small custom tool wrapping a Workflows webhook or Graph sendMail is the more predictable choice. **Apprise** supports a `workflows://` scheme for Teams Workflows; its `msteams://` scheme targets the retired connectors. Alert hygiene comes down to: dedup keys, throttling and backoff (the 4 req/s webhook cap, 429s), severity levels, actionable content with runbook links, and "resolved" follow-ups.

### Cited Findings
- Agent 365 / Copilot Studio Teams MCP reference: server ID `mcp_TeamsServer`, scope `McpServers.Teams.All`, URL `https://agent365.svc.cloud.microsoft/agents/tenants/{tenantId}/servers/mcp_TeamsServer`. Tools include `mcp_graph_chat_postMessage` (POST /v1.0/chats/{chat-id}/messages: "only text supported ... Not idempotent—avoid blind retries"), `mcp_graph_chat_listChats`, `mcp_graph_teams_postChannelMessage` ("Returns a 403 without ChannelMessage.Send"), and `mcp_graph_teams_replyToChannelMessage`. The feature is marked "preview ... aren't meant for production use", and "Application-specific Microsoft 365 MCP tools (... Mail ... Teams ...) are only intended for backward compatibility ... For new projects ... use Work IQ MCP." — [Learn: Teams reference (preview), updated Sep 2026](https://learn.microsoft.com/en-us/microsoft-agent-365/mcp-server-reference/teams); [Agent 365 MCP server options](https://learn.microsoft.com/en-us/microsoft-agent-365/tooling-servers-overview)
- Third-party reviews say the Work IQ Teams MCP requires an M365 Copilot license plus an admin-registered and consented Entra app. They also note community Teams MCP servers predate the official one. — [usecarly.com](https://www.usecarly.com/blog/microsoft-teams-mcp/); [ChatForest review](https://chatforest.com/reviews/teams-mcp-servers/); [Strac](https://www.strac.io/blog/microsoft-teams-mcp-server)
- Apprise: the legacy `msteams://{team}/{tokenA}/{tokenB}/{tokenC}/` scheme is for the retired Incoming Webhook (text-only, 1,000-character body). — [Apprise msteams](https://appriseit.com/services/msteams/). The Workflows scheme is `workflows://{host}:{port}/{workflow}/{signature}`, built from the Teams Workflows URL. — [optrix mirror of Apprise docs](https://docs.optrix.com.au/apprise:microsoft_teams_notifications) (not confirmed on appriseit.com)
- Microsoft's notification guidance for proactive messages: say what happened, what the result was, who or what triggered it, what users can do in response, and how to opt out. Track message IDs so you can update or delete messages later. — [Learn: proactive messages best practices](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/conversations/send-proactive-messages)
- Microsoft's terms forbid using Teams as a log file ("Only send messages that people will read"). — [Learn chat-post-messages](https://learn.microsoft.com/en-us/graph/api/chat-post-messages?view=graph-rest-1.0)
- Webhook rate cap: more than 4 requests per second triggers throttling, so use exponential backoff and honor 429. Payloads over 28 KB are rejected. — [Learn](https://learn.microsoft.com/en-us/microsoftteams/platform/webhooks-and-connectors/how-to/add-incoming-webhook)

### Inferences
- Alert-hygiene design for an OMEGAMON/RMF III AI monitor:
  - **Dedup key** (e.g. `sysplex:system:subsystem:metric:condition`). Suppress repeats within a window such as 15–30 minutes, and send "still firing (N occurrences)" digests instead of new posts.
  - **Severity mapping**: CRITICAL/HIGH alert both Teams and email, WARNING goes to Teams only, INFO goes to a daily digest.
  - **Rate limiting**: a token bucket of ≤1 msg/s per Teams destination, well under the 4/s cap, plus a global circuit breaker during alert storms that sends one roll-up message.
  - **Content**: what, where, when (UTC plus local time), current value vs threshold, likely cause (the AI's explanation, labeled as such), **runbook URL**, dashboard link, and owner.
  - **Resolution**: a "RESOLVED" follow-up, or a message edit via the bot API, when the condition clears.
  - **Delivery checks**: idempotency keys, since the Graph chat post and MCP postMessage are not idempotent. Log every send attempt, and fail over to the second channel (email) if Teams fails.
  - **Secrets**: keep webhook URLs and client secrets or certs in a vault, and use certificate auth for Entra apps.
- Have humans approve any AI-initiated escalations to broad audiences. Restrict an MCP-equipped agent to send-only tools for known chat and channel IDs to limit prompt-injection blast radius.

### Gaps
- I did not verify whether a Microsoft-official **Mail/Outlook** MCP server (Work IQ Mail) supports sending as an app identity, or its licensing details.
- I did not confirm the exact Apprise version that added `workflows://` (or an `office365://` email plugin for Graph) against the official Apprise wiki.
- I found no authoritative vendor source with quantitative dedup/throttling recommendations. The hygiene guidance above is synthesized practice, not cited standards.
