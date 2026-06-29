# macOS Desktop Agentic Tools for the Office Worker
## A Sophisticated Comparison: Manus, Claude Desktop (Chat · Cowork · Code), and OpenAI Codex

**Prepared by Manus AI · June 28, 2026**

---

## Preface

This report evaluates three leading macOS desktop AI agent platforms from the perspective of a **regular office worker** in an EU-based company. The evaluation explicitly excludes agentic software development as a use case and focuses instead on the practical, day-to-day workflows that define knowledge work: drafting documents, building and analysing spreadsheets, creating presentations, processing data from CSV and web sources, organising local files, transcribing meeting audio, writing meeting memos, and integrating with enterprise collaboration tools such as Atlassian Jira, Confluence, and Slack.

A central constraint throughout this analysis is **GDPR compliance and data sovereignty**. The company's security posture requires that corporate files remain local, on company servers, or in Microsoft SharePoint — not uploaded to third-party AI infrastructure without explicit data processing agreements and appropriate safeguards.

The three platforms compared are:

- **Manus** (v1.6.2, Meta/Monica AI, macOS arm64)
- **Claude Desktop** (v1.2.234, Anthropic), encompassing three distinct modes: **Chat**, **Cowork**, and **Code**
- **OpenAI Codex** (v26.616.81150, OpenAI, macOS arm64)

---

## 1. Platform Architectures: What Runs Where

Understanding the underlying architecture of each platform is essential for evaluating data privacy. The critical question for a GDPR-constrained organisation is not merely "can this tool do the task?" but "where does the data go when it does?"

### 1.1 Manus

Manus is a three-layer architecture: an **Electron 34.5.7 shell** provides the desktop UI; a **statically exported Next.js/React web application** handles nearly all user-facing functionality by communicating directly with Manus cloud APIs at `api.manus.im`; and a **15 MB Go sidecar** process provides the "My Computer" capability [^1].

The decisive architectural fact for privacy is that **AI inference and agent reasoning do not run locally**. The sidecar is a remote procedure channel: it receives instructions from the Manus cloud control plane, executes file system operations and terminal commands on the user's machine within the selected shared folder, and returns results to the cloud [^1]. When Manus reads a local spreadsheet to analyse it, the content of that spreadsheet is transmitted to Manus's cloud infrastructure for the AI to reason about. The sidecar connects outward via a signed WebSocket URL, and the cloud orchestrates what it does locally [^1].

Manus's infrastructure relies on US-based cloud providers: Amazon Web Services, Google Cloud Platform, and Microsoft Azure [^2]. The company holds SOC 2 Type II and ISO 27001 certifications [^2].

### 1.2 Claude Desktop: Three Modes, Three Architectures

Claude Desktop is architecturally the most complex of the three, because it bundles three fundamentally different execution environments under one application shell (Electron 40.8.5) [^3].

**Chat mode** is the simplest: the desktop app is essentially a webview wrapper loading `https://claude.ai`. All AI execution happens on Anthropic's servers inside a gVisor container (Google's synthetic kernel, running on `x86_64` server hardware). The user's local machine contributes nothing to execution; files uploaded to Chat are sent to Anthropic's infrastructure [^3].

**Cowork mode** is architecturally the most significant for office workers. It runs a full **Ubuntu 22.04 Linux virtual machine locally on the user's Mac** using Apple's native `Virtualization.framework` — no Docker, no QEMU, no third-party hypervisor [^3]. The VM uses a real Linux kernel (6.8.0, `aarch64`), and user-selected host folders are mounted into the VM via VirtioFS + FUSE. Critically, when Cowork reads and organises local files, **the files are processed inside the local VM and do not leave the machine** [^4] [^5]. The AI model still runs on Anthropic's servers, but the execution of file operations is local. The VM has a per-session domain allowlist enforced by a gVisor userspace network stack, preventing unrestricted internet access from within the VM [^3].

**Code mode** (Claude Code) is a developer-oriented CLI tool. It is explicitly out of scope for this evaluation per the user's requirements.

### 1.3 OpenAI Codex

Codex is a **native Rust app-server** bundled inside an Electron 42.1.0 shell [^6]. Unlike Claude's VM-based approach, Codex executes commands directly on the host macOS operating system, relying on an **approval policy** and a workspace-scoped sandbox rather than a hypervisor [^6] [^7]. The agent reasoning runs on OpenAI's servers (GPT-5.2-Codex and related models). Local file operations are mediated by the Rust backend, which enforces write permissions limited to the active workspace by default [^7].

Codex also includes an optional feature called **Chronicle**, which takes periodic screenshots of the user's desktop, sends them to OpenAI's servers for OCR processing, and stores text summaries locally as unencrypted Markdown files. Due to its fundamental incompatibility with GDPR's data minimisation and purpose limitation principles, Chronicle is **completely unavailable in the EU, UK, and Switzerland** [^8].

---

## 2. Capability Assessment by Office Use Case

The following sections evaluate each platform against the specific workflows identified for the target office worker.

### 2.1 Document Creation

| Capability | Manus | Claude Cowork | Codex |
|---|---|---|---|
| **Generate Word/DOCX reports** | Yes, natively | Via local Python script in VM | Via local script |
| **Generate PDF reports** | Yes, natively | Via local tools (e.g., `pandoc`) | Via local tools |
| **Generate PowerPoint/PPTX** | Yes, natively | Via local Python/`python-pptx` | Via local script |
| **Draft from multiple source files** | Yes | Yes (strong capability) | Yes |
| **Web research + document synthesis** | Yes (built-in browser) | Yes (web search tool) | Yes (in-app browser) |
| **Non-technical user experience** | Excellent | Good | Moderate |

Manus has the clearest advantage in document creation for non-technical users, as it natively outputs Office-compatible file formats without requiring the user to understand that a Python script must be written and executed [^9]. Claude Cowork is highly capable but typically produces Markdown or plain text, then uses a local script to convert to the final format — a process that is transparent to the user but adds a step [^10]. Codex follows a similar pattern.

### 2.2 Spreadsheet and Data Analysis

| Capability | Manus | Claude Cowork | Codex |
|---|---|---|---|
| **Read and analyse local CSV/Excel** | Yes (via My Computer) | Yes (via VM file access) | Yes (via workspace) |
| **Generate new Excel files with formulas** | Yes (with known reliability issues) | Via `openpyxl`/`xlsxwriter` in VM | Via local script |
| **Create charts and visualisations** | Yes | Yes (matplotlib, plotly) | Yes |
| **Interactive dashboards** | Limited | Via local HTML/Python | Yes (strong capability) |
| **Analyse data from web sources** | Yes | Yes | Yes |
| **Handle large datasets (>100k rows)** | Moderate | Good (local compute) | Good (local compute) |

Claude Cowork and Codex have a structural advantage for large-dataset analysis because they can run Python locally with full memory access, whereas Manus must transmit data to the cloud for reasoning. There are documented cases of Manus delivering blank or corrupted Excel files [^11], suggesting reliability issues with complex spreadsheet generation that Anthropic's local VM approach avoids.

### 2.3 Presentation Creation

All three platforms can generate slide decks. Manus offers the most polished native output, with a dedicated presentation mode and a "Presenter Window" in the desktop app [^1]. Claude Cowork can generate presentations via Python libraries but lacks a native presentation viewer. Codex can generate HTML-based presentations or use local tools.

### 2.4 Local File Organisation

| Capability | Manus | Claude Cowork | Codex |
|---|---|---|---|
| **Rename files in bulk** | Yes (terminal commands) | Yes (local VM) | Yes (workspace) |
| **Sort and categorise by content** | Yes (AI-driven) | Yes (AI-driven) | Yes |
| **Deduplicate files** | Yes | Yes | Yes |
| **Requires explicit user approval** | Yes (per command) | Yes (per action) | Yes (per policy) |
| **Files leave the local machine** | Yes (content for reasoning) | No (execution is local) | No (execution is local) |

The critical distinction is in the last row. When Manus organises files, the file names and content must be sent to the cloud for the AI to understand what to do. When Claude Cowork organises files, the AI model sends instructions to the local VM, which executes them — the files themselves remain on the machine [^4] [^5]. Codex similarly executes locally, though without the VM isolation layer.

### 2.5 Meeting Transcription and Memo Creation

This is a use case where the platforms diverge most sharply.

**Manus** has a clear, built-in advantage. It natively supports audio transcription from MP3, WAV, M4A, and WEBM files with high accuracy, even with accents and multiple speakers [^12]. A dedicated "AI meeting notes generator" workflow converts a recording into a structured transcript, executive summary, action items, and follow-up email draft in a single operation [^13]. For an office worker who regularly needs to turn recorded meetings into memos, Manus requires no technical knowledge and no additional tools.

**Claude Cowork** can perform audio transcription, but it does so by writing and executing a Python script inside the local VM that calls an external transcription API (such as OpenAI's Whisper). This means the audio file is sent to a third-party service, introducing an additional data flow that must be evaluated for GDPR compliance. The user experience is less seamless than Manus.

**Codex** faces the same limitation as Claude Cowork. It can write scripts to call transcription APIs, but there is no native, built-in audio processing pipeline. The audio file would need to be sent to an external service.

> **GDPR Note on Audio Transcription:** Meeting recordings frequently contain personal data (names, voices, discussed personal information). Under GDPR, sending such recordings to a third-party cloud service requires a valid legal basis and a Data Processing Agreement. Manus's built-in transcription sends the audio to Manus's cloud infrastructure; Claude's approach sends it to a separate API provider. Both require DPAs. The safest approach for a GDPR-constrained organisation is to use a locally running transcription model (e.g., OpenAI Whisper running locally), which none of these platforms currently offer natively.

### 2.6 Creating Interactive Analysis Reports

All three platforms can generate interactive HTML reports with charts and data visualisations. Codex has the strongest native capability here, given its origins as a developer tool and its ability to generate sophisticated web-based dashboards [^14]. Claude Cowork is equally capable, using Python (matplotlib, plotly, Dash) within its local VM. Manus can generate interactive reports but is more oriented toward polished static outputs.

---

## 3. MCP Integrations: Atlassian, Slack, and SharePoint

The Model Context Protocol (MCP) has become the standard integration layer between AI agents and enterprise data sources. All three platforms support MCP, but the maturity and ease of configuration differ.

### 3.1 Atlassian Jira and Confluence

Atlassian released an official **Remote MCP Server** (generally available from February 2026) that acts as a secure proxy between AI clients and Atlassian Cloud [^15]. The server does not store or cache Jira or Confluence content; it operates within the permissions of the signed-in user [^15]. Rate limits apply based on plan tier (500–10,000 calls/hour) [^15].

**Claude Desktop** is the primary launch partner for the Atlassian MCP server, and the integration is the most mature and well-documented [^16] [^17]. Desktop Extensions (`.dxt` files) allow one-click installation of MCP servers directly from within the Claude Desktop app, making this accessible to non-technical users [^18].

**Codex** supports MCP and can integrate with the Atlassian server via third-party tools like Composio [^19]. The configuration requires more technical knowledge than Claude's one-click approach.

**Manus** supports MCP via its Connector Registry. The integration with Atlassian is supported, but documentation and community resources are less extensive than for Claude [^20].

### 3.2 Slack

Slack provides an official MCP server that enables AI agents to search channels, read direct messages, and access shared files [^21]. All three platforms can connect to this server, but a Slack workspace admin must explicitly approve the integration before any user can connect [^21].

The Harmonic Security guide for enterprise Claude deployments notes that two distinct Slack integrations exist: **Claude as a Slack bot** (a chat assistant within Slack) and the **Slack MCP connector** (which reads Slack workspace data from within a Cowork session). These must be managed separately by enterprise administrators [^5].

### 3.3 Microsoft SharePoint

Community-built MCP servers for SharePoint exist, using the Microsoft Graph API to provide document operations (upload, download, search) and folder management [^22]. These work with all three platforms. Manus has announced deeper Microsoft ecosystem integration through Agent 365, providing administrators with visibility into tool access via Microsoft Purview, Defender, and Intune [^23].

| Integration | Claude Desktop | Codex | Manus |
|---|---|---|---|
| **Atlassian Jira/Confluence** | Excellent (official, one-click) | Good (via third-party) | Good (Connector Registry) |
| **Slack** | Excellent (official MCP + bot) | Good (MCP) | Good (MCP + native integration) |
| **Microsoft SharePoint** | Good (community MCP) | Good (community MCP) | Good (Agent 365 / MCP) |
| **Google Drive/Workspace** | Good (connector) | Moderate | Good (native integration) |
| **MCP ease of setup (non-technical)** | Excellent (one-click .dxt) | Moderate (config files) | Good (Connector Registry UI) |

---

## 4. GDPR, Data Privacy, and the EU Office Worker

This section provides the most critical analysis for the target organisation. The evaluation framework considers four dimensions: **where data is processed**, **where data is stored**, **what contractual protections exist**, and **what the practical risks are**.

### 4.1 The Fundamental Data Flow Problem

Every AI agent in this comparison relies on a cloud-hosted language model for reasoning. This means that for the AI to understand and act on a task, some representation of the task context — including file names, file contents, or descriptions of data — must be transmitted to the cloud. The architectural differences between the platforms determine *how much* data leaves the local machine and *when*.

```
┌─────────────────────────────────────────────────────────────────┐
│  Data Exposure Spectrum                                          │
│                                                                  │
│  MOST LOCAL                                    MOST CLOUD        │
│  ◄──────────────────────────────────────────────────────────►   │
│                                                                  │
│  Claude Cowork         Codex              Manus Chat/Cloud       │
│  (files in local VM,   (files in local    (files transmitted     │
│   only descriptions    workspace, only    to cloud for           │
│   sent to cloud)       descriptions       AI reasoning)          │
│                        sent to cloud)                            │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Claude Desktop: Data Residency and Compliance Status

Anthropic's current data residency situation is nuanced and important to understand correctly.

For **consumer products** (Claude Free, Pro, Max), including when those accounts use Claude Code or Cowork, **there is no EU data residency option** [^24]. Data is stored in the US. This is a significant barrier for EU companies whose employees use personal Claude subscriptions.

For **commercial products** (Claude Enterprise, API), Anthropic offers traffic routing to EU regions (via AWS and GCP European infrastructure) and provides a **Data Processing Addendum (DPA)** [^25]. However, even with EU routing, data is ultimately stored in the US [^25]. The DPA provides contractual protections under GDPR's standard contractual clauses (SCCs) framework.

A critical enterprise risk identified by security researchers is that **Cowork activity is currently excluded from Anthropic's Audit Logs, Compliance API, and Data Exports** [^26]. This means that an enterprise administrator cannot centrally monitor what files Claude Cowork has accessed or what actions it has taken — a significant gap for regulated environments that require comprehensive audit trails.

### 4.3 OpenAI Codex: EU Data Residency Available

OpenAI is currently the only platform in this comparison offering a formal **EU data residency** option, available to Enterprise, Edu, and API customers [^27] [^28]. With EU data residency enabled, data at rest is stored within the EU. OpenAI provides a DPA to support GDPR compliance [^29].

The **Chronicle** feature's complete unavailability in the EU is both a red flag and a reassurance. It is a red flag because it demonstrates that OpenAI itself recognises the feature is incompatible with GDPR — a company that truly understood GDPR would have designed it differently from the start. It is a reassurance because EU users are protected from it [^8].

Codex's lack of a macOS App Sandbox entitlement is a security consideration: the application relies on hardened runtime, code signing, renderer isolation, and its own approval policy rather than OS-level sandboxing [^6]. This is a common pattern for developer tools but means the security boundary is less strict than Claude's VM-based approach.

### 4.4 Manus: Compliance Certifications vs. Practical Gaps

Manus holds SOC 2 Type II and ISO 27001 certifications, which are meaningful indicators of security maturity [^2]. However, independent privacy analyses give Manus a "Grade C" on data trust indices, noting that while it discloses GDPR-grade data subject rights and legal basis mappings, it lacks a dedicated EU data residency option [^30].

The "My Computer" sidecar architecture means that when Manus works with local files, the **content of those files is transmitted to Manus's cloud infrastructure** for AI reasoning. This is architecturally different from Claude Cowork, where file content stays within the local VM and only task instructions and results cross the network boundary [^1].

Manus's telemetry footprint is also notable: the renderer includes Sentry, Amplitude, Intercom, and FingerprintJS-style analytics, and the sidecar includes Sentry [^1]. The exact data transmitted in these analytics payloads was not established by static analysis alone, but their presence is relevant for GDPR's data minimisation principle.

### 4.5 GDPR Compliance Summary Table

| Dimension | Claude Desktop (Enterprise) | Codex (Enterprise) | Manus (Team/Enterprise) |
|---|---|---|---|
| **EU Data Residency (data at rest)** | No (US storage) | Yes (available) | No (US infrastructure) |
| **EU Traffic Routing** | Yes (API/Enterprise) | Yes (Enterprise) | Not confirmed |
| **Data Processing Addendum** | Yes | Yes | Not publicly documented |
| **No training on business data (default)** | Yes | Yes | Yes |
| **SOC 2 Type II** | Yes | Yes | Yes |
| **ISO 27001** | Yes | Yes | Yes |
| **Audit Logs for AI actions** | No (Cowork excluded) | Partial | Not confirmed |
| **Local file execution (no upload)** | Yes (Cowork VM) | Yes (workspace) | No (cloud reasoning) |
| **Chronicle/screen capture** | No | EU-blocked | No |
| **GDPR risk level for EU company** | Medium (with Enterprise DPA) | Medium (with EU residency) | High (without DPA) |

---

## 5. Security Architecture Deep Dive

### 5.1 Claude Desktop's Layered Sandbox

Claude Cowork implements the most rigorous isolation architecture of the three platforms. Three nested layers protect the host system [^3]:

1. **macOS Seatbelt** (`sandbox-exec`) at the host level
2. **Apple Virtualization.framework** (a full Ubuntu 22.04 VM)
3. **bubblewrap + seccomp** per-process isolation within the VM

The VM's network access is restricted to an allowlist of domains enforced by a gVisor userspace TCP/IP stack. Before executing code, the app runs it through a Claude-powered safety classifier [^3]. User-selected folders are mounted with explicit permission modes (`ro`, `rw`, `rwd`), and a plugin permission bridge requires host-side approval for sensitive operations [^3].

This architecture means that even if a malicious document were to attempt a prompt injection attack, the VM sandbox limits what damage can be done. The VM cannot access the broader host filesystem beyond explicitly shared folders.

### 5.2 Codex's Approval-Based Security

Codex relies on a **two-layer security model**: a sandbox mode (limiting write access to the active workspace) and an approval policy (requiring user confirmation before sensitive actions) [^7]. By default, network access is turned off for agent-executed commands, and write permissions are limited to the current workspace [^7].

The absence of a macOS App Sandbox entitlement means the application has broader system access than a sandboxed app, but this is mitigated by the Rust app-server's own policy enforcement [^6]. The approval model is effective for technical users who understand what they are approving, but may be less protective for non-technical office workers who may reflexively approve prompts without reading them carefully.

### 5.3 Manus's Cloud-Orchestrated Trust Model

Manus's security model is fundamentally different: the trust boundary is not a local sandbox but a **cryptographically signed WebSocket connection** between the local sidecar and the cloud control plane [^1]. The sidecar verifies the signed URL before connecting, preventing a compromised control plane from redirecting the sidecar to an arbitrary endpoint [^1].

However, the sidecar starts a real local PTY (terminal) and shell, inheriting the app's environment. A short denylist of destructive commands (e.g., `rm -rf /`) provides guardrails, but this is not a general sandbox — the sidecar can exercise the full permissions of the user's macOS account within the shared folder [^1]. The primary scope boundary is the selected shared folder plus protocol validation.

---

## 6. User Experience and Accessibility for Non-Technical Workers

The target user is an office worker without a technical background. Ease of use is therefore a critical evaluation dimension.

| Dimension | Manus | Claude Cowork | Codex |
|---|---|---|---|
| **Initial setup complexity** | Low (download, sign in) | Low (download, sign in) | Low (download, sign in) |
| **MCP/integration setup** | Medium (Connector Registry UI) | Low (one-click .dxt extensions) | Medium (config files) |
| **Giving a task in plain language** | Excellent | Excellent | Good |
| **Understanding what the agent is doing** | Good (approval prompts) | Good (shows actions) | Good (approval prompts) |
| **Output quality for non-technical tasks** | Excellent | Excellent | Good |
| **Audio transcription (no setup)** | Yes (native) | No (requires script) | No (requires script) |
| **Scheduling recurring tasks** | Yes | Yes | Yes |
| **Mobile companion app** | Yes (iOS/Android) | No | No |

Manus has the clearest advantage for non-technical users in terms of out-of-the-box productivity. Its native audio transcription, direct Office-format output, and mobile companion app make it the most accessible platform for the described office worker profile. Claude Cowork is a close second, with its one-click MCP extension store making enterprise integrations more accessible than Codex.

---

## 7. Pricing Overview

| Plan | Manus | Claude Desktop | Codex |
|---|---|---|---|
| **Free tier** | Yes (limited credits) | Yes (limited) | Yes (limited) |
| **Individual paid** | ~$20–40/month (credit-based) | Pro $20/mo, Max $100–200/mo | Plus $20/mo, Pro $100/mo |
| **Team** | $20/seat/month | $20–25/seat/month (standard) | Business plan (contact) |
| **Enterprise** | Contact sales | Contact sales | Contact sales |
| **Cowork/agentic features included** | Yes (all plans) | Pro and above | All plans |

All three platforms use a credit or usage-based model at higher tiers. Claude's pricing is the most transparent, with clearly defined usage multipliers (5x, 20x) for Max plans.

---

## 8. Recommendations for an EU-Based Office Worker

### 8.1 For Strict Local File Privacy (GDPR Priority)

**Recommended: Claude Desktop on an Enterprise Plan with DPA**

Claude Cowork's local VM architecture is the only approach in this comparison where file content genuinely does not leave the user's machine during execution. For an EU company that handles sensitive documents (HR records, financial data, legal contracts), this is the most defensible architecture. The organisation must:

1. Execute Anthropic's Data Processing Addendum before deployment.
2. Enable EU traffic routing for the API/Enterprise plan.
3. Acknowledge that data is still stored in the US (mitigated by SCCs in the DPA).
4. Implement enterprise controls to disable Cowork for regulated use cases until Anthropic adds Cowork to its Audit Logs.

### 8.2 For EU Data Residency Compliance

**Recommended: OpenAI Codex Enterprise with EU Data Residency**

If the organisation's legal team requires data at rest to be stored within the EU, Codex Enterprise with EU data residency enabled is currently the only option that satisfies this requirement. The organisation should ensure Chronicle is disabled (it is blocked in the EU by default) and configure the workspace sandbox appropriately.

### 8.3 For Meeting Transcription and Memo Creation

**Recommended: Manus (with DPA and awareness of data flows)**

For the specific use case of transcribing MP3 meeting recordings and generating memos, Manus provides the most seamless, non-technical experience. However, the organisation must be aware that audio content is transmitted to Manus's cloud infrastructure for processing. A DPA with Manus is required, and the organisation should assess whether the audio recordings contain personal data that requires additional safeguards.

### 8.4 For Atlassian and Slack Integrations

**Recommended: Claude Desktop (best MCP ecosystem) or Manus (native Slack integration)**

Both Claude Desktop and Manus offer mature integrations with Atlassian and Slack. Claude's one-click Desktop Extensions make the setup most accessible for non-technical users. The Atlassian Remote MCP Server itself does not store Jira or Confluence content [^15], making it a relatively safe integration from a data perspective — the data flows from Atlassian to the AI agent's context window, not to a new storage location.

### 8.5 For SharePoint-Centric Organisations

**Recommended: Manus (Agent 365 integration) or Claude Desktop (SharePoint MCP)**

Manus's announced Agent 365 integration provides the deepest Microsoft ecosystem alignment, including visibility via Microsoft Purview [^23]. For organisations already heavily invested in Microsoft 365, this makes Manus the most strategically aligned choice, provided the DPA and data residency concerns are addressed.

---

## 9. Summary Scorecard

The following scorecard rates each platform on a five-point scale (1 = poor, 5 = excellent) for each evaluation dimension relevant to the target office worker.

| Evaluation Dimension | Manus | Claude Cowork | Codex |
|---|---|---|---|
| **Document creation (non-technical)** | 5 | 4 | 3 |
| **Spreadsheet analysis** | 3 | 4 | 4 |
| **Presentation creation** | 5 | 3 | 3 |
| **Local file organisation** | 4 | 5 | 4 |
| **Meeting transcription (native)** | 5 | 2 | 2 |
| **Meeting memo generation** | 5 | 4 | 3 |
| **Interactive analysis reports** | 3 | 4 | 5 |
| **Atlassian MCP integration** | 3 | 5 | 3 |
| **Slack integration** | 4 | 4 | 3 |
| **SharePoint integration** | 4 | 3 | 3 |
| **GDPR / data privacy (EU)** | 2 | 3 | 4 |
| **Local file execution (no upload)** | 2 | 5 | 4 |
| **EU data residency** | 1 | 2 | 5 |
| **Audit logging for enterprise** | 2 | 2 | 3 |
| **Non-technical user experience** | 5 | 4 | 3 |
| **MCP setup ease** | 3 | 5 | 3 |
| **Pricing transparency** | 3 | 4 | 4 |
| **TOTAL (out of 85)** | **59** | **63** | **60** |

> **Note:** Scores reflect the current state as of June 2026 and will change as platforms evolve. Claude Cowork's lead is primarily driven by its superior local execution architecture and MCP ecosystem. Manus leads on user experience and native multimedia capabilities. Codex leads on EU data residency.

---

## 10. Closing Observations

The three platforms represent different philosophies about where AI agents should live. Manus is a cloud-first agent that reaches into the local machine when needed — maximally capable but maximally exposed. Claude Cowork is a local-first executor with cloud intelligence — the most privacy-preserving architecture for file work, but constrained by Anthropic's US-centric data infrastructure. Codex is a local developer tool that has expanded into office work — technically capable but less polished for non-technical users, and the only platform with a genuine EU data residency story.

For the EU-based office worker described in this report, **no single platform is a perfect fit today**. The pragmatic path forward is a layered strategy: use **Claude Cowork on an Enterprise plan** for local file manipulation and document synthesis; use **Atlassian's official Remote MCP Server** (compatible with all three platforms) for Jira and Confluence access, keeping that data within Atlassian's own infrastructure; use **Manus** for meeting transcription and memo generation, with a DPA in place; and evaluate **Codex Enterprise with EU data residency** as the long-term foundation once Anthropic's audit logging gaps are addressed.

The most important principle for GDPR compliance is not which tool you choose, but **how you configure it**: disable screen recording features, execute DPAs before processing personal data, restrict agent access to explicitly shared folders, and prefer MCP-based integrations (which access data through existing permission systems) over direct file uploads to AI cloud services.

---

## References

[^1]: Project File: `manus-research-by-codex.md` — Codex-authored static analysis of Manus.app v1.6.2
[^2]: Manus Trust Center. https://trust.manus.im/
[^3]: Project File: `research-claude-desktop-virtualization.md` — Detailed virtualization and sandboxing research for Claude Desktop v1.2.234
[^4]: Substack. "I tried using Claude Cowork to organize 2,200 files in my Downloads folder". https://wonderingaboutai.substack.com/p/i-tried-using-claude-cowork-to-organize
[^5]: Harmonic Security. "Securing Claude Cowork: A Security Practitioner's Guide". https://www.harmonic.security/resources/securing-claude-cowork-a-security-practitioners-guide
[^6]: Project File: `codex-research-by-codex.md` — Static analysis of Codex.app v26.616.81150
[^7]: OpenAI Developers. "Agent approvals & security – Codex". https://developers.openai.com/codex/agent-approvals-security
[^8]: The Next Web. "OpenAI's Codex for Mac now watches your screen to build context, but sends the screenshots to its servers first". https://thenextweb.com/news/openai-codex-chronicle-screen-context-mac
[^9]: Substack. "Manus AI 101: The Complete Guide to the Autonomous AI Agent". https://sidsaladi.substack.com/p/manus-ai-101-the-complete-guide-to
[^10]: Substack. "Claude Cowork Starter Guide + 30 examples". https://claudiaplusai.substack.com/p/claude-cowork-starter-guide-30-examples
[^11]: Reddit. "Manus delivering blank excel files". https://www.reddit.com/r/ManusOfficial/comments/1l50r2b/manus_delivering_blank_excel_files/
[^12]: Manus Documentation. "Multimedia Processing". https://manus.im/docs/features/multi-modal
[^13]: Manus. "AI meeting notes generator". https://manus.im/playbook/ai-meeting-notes
[^14]: YouTube. "I Turned a CSV into an Interactive Dashboard with OpenAI Codex". https://www.youtube.com/watch?v=NJCwjZ75RYg
[^15]: Atlassian. "Extend Atlassian into any AI assistant using MCP". https://www.atlassian.com/platform/remote-mcp-server
[^16]: MindStudio. "Atlassian's MCP Server Is Now GA: How Claude Reads and Writes Jira, Confluence, Compass". https://www.mindstudio.ai/blog/atlassian-mcp-server-ga-claude-reads-writes-jira-confluence-compass-oauth
[^17]: Atlassian. "Introducing Atlassian's Remote Model Context Protocol (MCP) Server". https://www.atlassian.com/blog/announcements/remote-mcp-server
[^18]: Anthropic Engineering. "Claude Desktop Extensions: One-click MCP server installation". https://www.anthropic.com/engineering/desktop-extensions
[^19]: Composio. "How to integrate Jira MCP with Codex". https://composio.dev/toolkits/jira/framework/codex
[^20]: LinkedIn. "Manus AI is different". https://www.linkedin.com/posts/jeremytang_manus-ai-is-different-it-marks-a-major-activity-7305339245630496768-SQON
[^21]: Slack Developer Docs. "Connect to Claude". https://docs.slack.dev/ai/slack-mcp-server/connect-to-claude
[^22]: Reddit. "SharePoint MCP Server – Provides Claude with access to Microsoft SharePoint". https://www.reddit.com/r/mcp/comments/1r8js0f/sharepoint_mcp_server_provides_claude_with_access/
[^23]: LinkedIn. "Microsoft announces Agent 365, integrating Manus AI". https://www.linkedin.com/posts/manus-im_msignite-agent365-manusai-activity-7396684625412104192-TFZg
[^24]: Reddit. "Anthropic, your fastest-growing region can't actually use Claude properly". https://www.reddit.com/r/ClaudeAI/comments/1rk06ou/anthropic_your_fastestgrowing_region_cant/
[^25]: Anthropic Privacy Center. "Where are your servers located? Do you host your models on EU servers?". https://privacy.claude.com/en/articles/7996890-where-are-your-servers-located-do-you-host-your-models-on-eu-servers
[^26]: TrueFoundry. "Claude Cowork Security Risks: The Enterprise Guide to Safe Deployment". https://www.truefoundry.com/blog/claude-cowork-security-risks
[^27]: OpenAI. "Expanding data residency access to business customers worldwide". https://openai.com/index/expanding-data-residency-access-to-business-customers-worldwide/
[^28]: OpenAI. "Introducing data residency in Europe". https://openai.com/index/introducing-data-residency-in-europe/
[^29]: OpenAI. "Enterprise privacy at OpenAI". https://openai.com/enterprise-privacy/
[^30]: VerifyWise. "Is Manus safe with your data? Grade C". https://verifywise.ai/ai-trust-index/manus
