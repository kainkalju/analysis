I'll research this thoroughly to give you a comprehensive comparison based on current product capabilities and licensing.A lot of these results are sales-funnel content from migration consultants (and most are angled toward M365, not Google — opposite of your direction). Let me get more specific, vendor-neutral technical detail on the areas that matter most for an M365 E5 → Google Workspace move: Outlook/Exchange→Gmail specifics, SharePoint/Teams→Drive specifics, security/compliance gaps, and Office file compatibility.Good, I have a clear picture now on the compliance/security gap. Let me now check the specific Excel/Office file fidelity and Teams-to-Meet/Chat migration angle, since these are the most concrete day-to-day pain points for 1,200 users.This confirms what I'd expect. One more check — Teams/SharePoint to Google Drive/Chat migration mechanics and the device management (Intune) gap, since with 1,200 users that's an operationally significant area.I have enough to write a grounded, practical answer. Most of these sources are vendor/migration-consultant content angled toward selling services (and the majority are actually pitching the opposite direction — M365 retention), so I'll synthesize the technical substance and flag where claims are marketing rather than fact. Given the length and structure of this, I'll put it in a document you can actually use internally.Given this is a "discuss/decide" question rather than a "give me a deliverable to send to a client" request, I'll answer it conversationally in chat rather than building a Word doc — that matches the guidance to keep analysis/strategy content inline. I'll offer the doc as a follow-up option.

## Short version

Going from M365 E5 (1,200 users) to Google Workspace Enterprise is doable and plenty of large orgs run on it — but you're trading a deeper, more integrated security/compliance/admin stack and best-in-class desktop Office for a simpler, browser-native, real-time-collaboration-first platform. The pain concentrates in four areas: **Outlook→Gmail migration mechanics**, **Excel fidelity loss**, **SharePoint/Teams→Drive/Chat restructuring**, and **the E5 security/compliance feature gap**. Below is what changes and what you'd likely miss, organized by the categories that matter at your scale.

---

## 1. Email & Calendar (Outlook/Exchange → Gmail/Calendar)

**What changes:**
- Mail flow rules, transport rules, and mail flow connectors in Exchange Online don't map 1:1 to Gmail's compliance rules — they get rebuilt, not migrated.
- Shared mailboxes become Google "delegated" or "group" mailboxes — different permission model.
- Distribution lists and dynamic distribution lists map to Google Groups, but dynamic membership rules need to be rebuilt.
- Gmail messages, folders, labels, and rules generally migrate cleanly, and calendar events, recurring meetings, shared calendars, room resources, and permissions also transfer with reasonably mature tooling in both directions.

**What you'd likely miss:**
- **Outlook desktop client behaviors** — offline access, complex rules, categories/color-coding, and some calendar delegate nuances that power users (especially EAs/executives) rely on heavily. Gmail/Calendar web and mobile are excellent, but Outlook desktop habits (especially around delegated calendar management for executives) are one of the most common complaint sources in this kind of switch.
- **Conditional Access at the mail layer** — Exchange Online lets you build very granular access/transport rules tied to Entra ID Conditional Access. Google's equivalent (context-aware access) is comparably capable for app access but less mature for mail-flow-specific routing logic.

---

## 2. Files: SharePoint/OneDrive → Drive/Shared Drives

This is usually the most labor-intensive part of the whole migration, and the structural model is genuinely different, not just renamed.

- SharePoint and Teams files move to Google Shared Drives via a native Google import tool, but lists, pages, automations, and Teams conversations require separate handling because they don't have a direct equivalent. SharePoint lists don't map directly to Drive — they have to become Sheets, AppSheet, or another tool depending on usage, and SharePoint pages either get rebuilt as Google Sites or archived.
- OneDrive's permission model differs from Drive's: Google's "Commenter" role has no native OneDrive equivalent, so commenter-access users get full Edit access by default after migration — the reverse direction has a similar problem (Drive's Viewer/Commenter/Editor doesn't map cleanly onto SharePoint's Read/Contribute/Edit roles), so you need an explicit permissions remap project, not an assumption that ACLs "just carry over."
- The recommended target for shared/team content is Shared Drives rather than individual My Drive, because Shared Drive content is owned by the team/org rather than a person — this avoids the classic problem of files disappearing or orphaning when someone leaves.

**What you'd likely miss:**
- **SharePoint metadata/managed columns, content types, and retention-at-the-library level.** Drive has labels but the metadata/taxonomy depth SharePoint offers (especially with custom content types and term stores) doesn't have a direct equivalent.
- **Power Automate / Power Apps built on top of SharePoint lists.** These get *redesigned*, not migrated — if any business unit has shadow-IT automation sitting on SharePoint lists or libraries, budget real time to rebuild this in Apps Script / AppSheet.
- **SharePoint site permission inheritance and granular library-level permissions** — Shared Drives' permission model is simpler (which many users will prefer) but less granular than what SharePoint admins are used to building.

---

## 3. Teams → Google Meet + Chat

This is probably the single biggest *cultural* and workflow change for 1,200 users, more than email or files.

- Moving from Teams to Google Chat/Meet is widely flagged as a major disruption point: Teams currently functions as a unified hub combining chat, video, phone system, and file collaboration in one application, while Google Meet is strong for video but is not positioned as a comparably deep collaboration hub.
- Google Chat history has no native export mechanism for individual users; admins can pull organizational chat data via Takeout or Vault before decommissioning, but there's no clean one-click "migrate my chat history" path — and the reverse is also messy (Teams chat history import into Google Chat is similarly limited).
- Teams "Channels" with persistent tabs (files, planner, wikis, third-party app tabs pinned into a channel) don't have a Chat equivalent — Chat Spaces are simpler.

**What you'd likely miss:**
- **Teams Phone**, if you use it for PSTN calling — Google Workspace doesn't have a direct telephony/PBX replacement; you'd need Google Voice (limited) or a third-party UCaaS provider.
- **Persistent channel structure with integrated tabs** (Planner, embedded SharePoint pages, third-party app tabs) — Chat Spaces are flatter and less "hub-like."
- **Meeting/webinar scale features** if you currently lean on Teams Town Hall/Webinar — verify Meet's current large-meeting and streaming caps against your actual use cases (live streaming/large town halls are supported in Workspace's higher tiers, but check participant caps for your specific scenarios).

---

## 4. Excel/Office files → Sheets/Docs/Slides

This is the area with the most consistently documented friction, so it's worth being concrete.

- VBA macros do not run in Sheets at all — Sheets uses Google Apps Script (a JavaScript-based language), and the two are not compatible; workbooks heavily dependent on macros are essentially Excel-only without a substantial rewrite.
- Power Query, Power Pivot, the Excel data model, LAMBDA, and some dynamic array behaviors have no equivalent in Sheets and either error out or get converted to static values on import.
- Complex conditional formatting, data bars/icon sets, cell styles/themes, and print layout (headers/footers/page breaks) often need manual rework after conversion, and shapes/SmartArt/embedded OLE objects are frequently lost or rasterized.
- Basic pivot tables convert, but pivot cache features, the data model, and relationships don't carry over, and calculated fields often need manual correction.
- Pivot tables transfer their structure but recalculate against the underlying data in Sheets — if your Excel pivots used the data model or external connections, the pivot results may come back empty or different, typically requiring a from-scratch rebuild.

**Bottom line on this point:** if you have finance, FP&A, or ops teams with heavy VBA-driven workbooks or Power Query/Power Pivot models, plan for a deliberate inventory-and-rebuild project — this won't migrate, it has to be re-engineered. For everyone else doing "normal" spreadsheet work (data entry, standard formulas, basic charts/pivots), the experience is genuinely good and friction is low.

---

## 5. Security, compliance, and admin — the real E5-specific gap

This is the part most relevant to you specifically, because you're coming *from* E5, which is Microsoft's top compliance/security tier.

- M365 E5 bundles Defender for Office 365 (advanced threat protection), Microsoft Purview (DLP, sensitivity labels, information barriers, insider risk management), Entra ID Conditional Access with a large set of signal conditions, and PIM for just-in-time admin access, plus eDiscovery Premium with legal hold, custodian management, review sets, and predictive coding.
- The Google equivalent stack: Google Workspace Enterprise Plus includes BeyondCorp Enterprise for zero trust, Google Vault for eDiscovery, and DLP capabilities, with roughly 40+ compliance certifications versus Microsoft's 100+. Vault and Purview are conceptually similar but not equivalent in depth: Purview lets you build policies with very fine-grained control — by content type, user, date, or metadata tags — while Vault's retention options are generally less granular, working at the org-unit or custom-scope level rather than with the same dynamic rule depth. Vault also covers Gmail, Drive, and Chat specifically, scoped to the Google ecosystem, whereas Purview spans Exchange, SharePoint, Teams, and OneDrive under one governance layer.
- On identity/device management specifically: Microsoft's native security tools are credited with reducing the need for third-party security add-ons by a meaningful margin in M365-centric environments. Practically, this means: Conditional Access policies, Intune-based device compliance, and Insider Risk Management are things you'd need to either replace with Google's Context-Aware Access + endpoint management + a third-party CASB/DLP tool, or accept a less granular native equivalent.

**What you'd likely miss, concretely:**
- **Microsoft Purview's cross-workload DLP and sensitivity labels** that travel with a file regardless of where it's stored/shared (Exchange, SharePoint, Teams, OneDrive all governed under one policy engine). Google's DLP is real but scoped per-service (Gmail DLP, Drive DLP) rather than one unified labeling layer.
- **Insider Risk Management and Communication Compliance** — no direct Google Workspace equivalent; if you use these today for regulated-role monitoring, you'd likely need a third-party tool.
- **eDiscovery sophistication** — Vault does search/hold well for Gmail/Drive/Chat, but lacks predictive coding, custodian management, and review-set workflows that Purview eDiscovery Premium provides. Worth validating directly against your legal/compliance team's actual requirements rather than assumption — if you're not doing heavy litigation holds today, this gap may not matter to you in practice.
- **Customer Lockbox** (Microsoft support staff access requiring your explicit approval) doesn't have a precisely equivalent Google control, though Google does offer Access Transparency logs at the Enterprise Plus tier as a partial analogue.
- **Compliance certification breadth** — if you operate in a regulated vertical (healthcare/finance/government/defense), check specific certifications you actually need (HITRUST, CJIS, ITAR, FedRAMP High vs Moderate) rather than relying on a general "fewer certs" framing — it matters only if your actual obligations require the specific ones that differ.

One important caveat: a large share of the content making these comparisons (including several of the consultancy sources above) is written by firms that sell M365 migration services, so the framing consistently favors Microsoft. The underlying facts I've cited (feature presence/absence, certification counts, granularity descriptions) are consistent across independent and vendor sources, but treat broad claims like "Microsoft is more secure" as commercial framing rather than a settled fact — the right test is which specific controls *your* compliance obligations actually require.

---

## 6. Identity & device management

- If you're using Entra ID-based Conditional Access tied to Intune device compliance today, that whole layer needs rebuilding on Google Context-Aware Access + Google endpoint management (or a third-party MDM/CASB).
- Any SSO/SAML/OIDC integrations currently pointed at Entra ID as the identity provider need to be reconfigured against Google Cloud Identity — this is real per-application work, not a flag flip, especially for 1,200 users' worth of connected SaaS apps.
- Windows-domain-joined device management (Group Policy, native AD integration) is a Microsoft-native strength; Google manages Windows devices but less natively than Intune/Entra does.

---

## 7. What you'd likely *gain*

To be even-handed, since you asked specifically what you'd "miss" — worth noting what generally improves:
- Real-time multi-user co-editing in Docs/Sheets/Slides is widely considered smoother than Office's co-authoring, with less merge-conflict friction.
- Generally lower admin overhead for day-to-day management — simpler licensing tiers, simpler sharing model once your org adjusts to it.
- User satisfaction with built-in AI is reported as higher among Workspace users than M365 users in some surveys (treat this as a self-reported survey data point, not a hard technical fact).
- Lower total license cost at list price, though several sources above argue the "true" cost converges once you add equivalent security/compliance tooling — that's a real consideration worth modeling for your specific stack, not taking at face value either way.

---

## Suggested next step for your specific situation

Given the scale (1,200 users, E5), the decision-relevant work is really:
1. **Inventory of Excel/VBA-dependent workbooks** by department — this alone often determines feasibility/timeline more than anything else.
2. **A line-by-line mapping of which E5 security/compliance controls you actually use today** (not just license, but actual configured policies) against Google Enterprise + whatever third-party tools would be needed to close gaps.
3. **SharePoint lists/Power Automate inventory** — anything beyond plain document libraries needs a redesign plan, not a migration plan.

Want me to put together a structured checklist or comparison table (e.g., as a working doc you can hand to IT/security/finance stakeholders) covering these gap areas against your specific E5 SKU entitlements?