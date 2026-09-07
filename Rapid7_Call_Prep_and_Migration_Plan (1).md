# Rapid7 SME Call – Question List & Migration Plan
*Supporting document to: Sentinel_DataLake_HLD.docx*

---

## 1. Questions for the Rapid7 Engineer

### A. Current onboarding / collection architecture
- Walk me through how a log source gets onboarded today – what's the standard process end-to-end?
- Is collection mostly agent-based, collector/relay-based, cloud-API pull, or a mix? Which method is used for which source types?
- Are there central collectors/relays (on-prem or cloud) we'd need to replace or repoint, or does everything go direct to InsightIDR?
- Is there an architecture diagram or config export we can get a copy of, even informally?

### B. Per-source onboarding detail
- For the **Cisco estate** (ISE, Secure Access/SSE suite, Stealthwatch, Umbrella, ESA) – is it all syslog/CEF, or does anything use a vendor-specific API?
- For **cloud sources** (Azure, AWS, GCP, O365) – native API integration or an intermediary (Lambda/Function/Event Hub)?
- The inventory shows duplicate-looking entries (`Office 365` and `Office365`, `Cisco Secure Access Pilot` and `Cisco Secure Acess Pilot`) – are these genuinely duplicate configs, typos in the sheet, or two distinct integrations?
- **GitHub** suite (Advanced Security, Audit Log, EMU variants, Enterprise) – is this webhook-based or scheduled API pull?

### C. Integration / forwarding methods available
- What forwarding methods does Rapid7 currently expose that we could reuse as a *bridge* during migration (e.g. can InsightIDR fan out / dual-forward to a second destination)?
- For sources with no native Sentinel connector, how is Rapid7 currently parsing/normalising them – custom parsers we could reference?
- Any sources that are hard-dependent on Rapid7-specific agents (Insight Agent) that would need an agent swap rather than a config change?

### D. Volume, schema, parsing
- Which sources are the highest-volume today (EPS / GB per day)? We need this to size the Sentinel Analytics vs. Data Lake split.
- Any sources with known parsing pain – unparsed/raw data, inconsistent schema, multi-format feeds?
- Are there existing field mappings / LEEF-CEF normalisation docs we could reuse instead of rebuilding from scratch?

### E. Retention & current usage
- What retention is configured per source today, and is it driven by a specific compliance requirement we need to carry over?
- Which sources are actually queried/used in investigations vs. collected "just in case"? (Helps confirm the Data Lake-tier candidates.)
- Any sources feeding scheduled reports, dashboards, or compliance evidence that we must not break during cutover?

### F. Specific open items from the inventory (need an answer/owner on each)
- **Absolute DDS** – confirm what "requires a server to pull the logs" means technically; can we reuse or do we need a new pull host?
- **Internal DNS Logs** – what does "MPZ to speak with Cloud" refer to; who owns this integration?
- **AWS Flow Logs** – what's outstanding on the Tamas conversation; is this a permissions/config issue or a design decision?
- **AWS Security Hub** – are findings actually deduplicated against GuardDuty today, or is this a known gap?
- **Azure Firewall / Azure Flow Logs** – where do these logs currently land (Storage Account, Event Hub, Log Analytics) – what needs "review"?
- **Cisco ISE** – what specifically needs review with the Networks team – device count, feed completeness?
- **Cisco Secure Access – China** – are logs flowing at all today? Any regional/data-residency constraint we should know about?
- **GCP Workspace Logs** – what configuration is outstanding with Steve?
- **ThreatCommand** – confirm this is being replaced (not migrated) – what's the replacement product and timeline?
- **McAfee / Trellix** – is this actively being onboarded, or should we deprioritise until Cloud team confirms config?
- **NetApps** – what's the issue with these logs specifically?
- **Service Now** – what configuration is outstanding with the ServiceNow team?
- **Sysmon** – how is this currently collected (Insight Agent, WEF, other), and does that collection method carry over?
- **U8 Finance Application Logs** – confirm root cause of "logs missing" – source-side or collector-side?
- **Web Access Log** – can you help us get the list of applications this is meant to cover?
- **Cisco AMP** – confirm this is fully decommissioned / being replaced, so we can drop it from scope entirely.

### G. Recommendations & constraints for Sentinel / Data Lake
- Based on what you've seen elsewhere, any sources that are notoriously painful to migrate that we should sequence later?
- Any Rapid7-specific enrichment (asset/user context, threat intel) we'd lose that Sentinel would need to replace separately?
- Any contractual/licensing constraints on running Rapid7 and Sentinel in parallel (e.g. dual-forwarding limits, data-out restrictions)?

### H. Dependencies & ownership
- Who are the right contacts for each source family (you mentioned Tamas, Steve, MPZ, Cloud team, Networks team, ServiceNow team) – can we get a proper contact list?
- Is there a Rapid7 support/renewal date that puts a hard deadline on this migration?
- Are there other consumers of Rapid7 data (beyond the SOC) we need to loop in – compliance, GRC, other tooling integrations?

---

## 2. Migration Steps (Phased Plan)

### Phase 0 – Discovery & Design (weeks 1–2)
1. Hold the Rapid7 SME working session (this call) and capture answers against §1 above.
2. Finalise Sentinel workspace design and Data Lake tier boundaries with the platform/architecture team.
3. Update the HLD (§5–§6) with confirmed volumes, integration methods, and tiering per source.
4. Confirm decommission scope (Cisco AMP, ThreatCommand replacement) and remove from onboarding backlog.

### Phase 1 – Foundation Build (weeks 2–4)
5. Stand up the Sentinel workspace(s) and Data Lake tier; configure RBAC, cost controls, and retention defaults.
6. Establish core connectivity: Azure Monitor Agent / Data Collection Rules, Event Hub namespaces, Logic App connectors as required.
7. Stand up parsing/normalisation (KQL functions) for sources without a native connector, using Rapid7 field mappings as a starting reference where available.

### Phase 2 – Wave 1: Tier 1 Critical Security Telemetry (weeks 4–8)
8. Onboard identity & cloud admin sources first (Azure, AWS, GCP, Office 365, Internal DNS) – highest detection value, native connectors available.
9. Onboard network & perimeter sources (Cisco ISE, Secure Access/SSE suite, Umbrella, Stealthwatch, Azure Firewall, flow logs, SilverPeak).
10. Onboard endpoint/identity sources (DHCP-QIP, Absolute DDS, Local Account/Service Creation, Sysmon).
11. Run Rapid7 and Sentinel **in parallel** for each source as it lands; validate event completeness and field parity before treating Rapid7 as backup-only for that source.
12. Rebuild/port priority detection rules from Rapid7 into Sentinel analytics rules for Wave 1 sources.

### Phase 3 – Validation (ongoing through Wave 1–2)
13. Compare alert volume and detection fidelity, Sentinel vs. Rapid7, for a minimum bake-in period (recommend 2–4 weeks per source) before sign-off.
14. Confirm dashboards/reports/compliance evidence that depended on Rapid7 have Sentinel equivalents.

### Phase 4 – Wave 2: Tier 2 + Known Gaps (weeks 8–12)
15. Close the "required but not flowing" gaps (Autodesk, Projectwise, ESRI, Atlassian, Axway) – these need configuration work, not just a platform swap.
16. Onboard DevSecOps/AppSec sources (GitHub suite, InsightAppSec suite) and remaining SaaS (Keeper, ServiceNow, Databricks).
17. Resolve the remaining open items from §1.F with the newly identified owners.

### Phase 5 – Wave 3: Net-New Gap Sources (weeks 12–20, business-driven pace)
18. Engage application owners for business/engineering platforms (Oracle suite, CRM, Aconex, and the internal ARUP systems) – route to Data Lake tier by default given lower real-time detection value.
19. Onboard remaining cloud-native/DevSecOps gaps (CloudTrail non-admin actions, CI/CD, SAST/DAST, API gateways, WAFs) as pipelines/owners are confirmed.

### Phase 6 – Cutover & Decommission (per source, once validated)
20. Once a source has passed parallel-run validation, disable duplicate collection in Rapid7 for that source (leave read-only access for a defined grace period).
21. Track cutover status per source against the Tier tables in the HLD (§6) so nothing is dropped mid-migration.
22. Once all in-scope sources are cut over and any compliance/evidence requirements are met, proceed to full Rapid7 decommission and contract exit (separate workstream, out of scope for this HLD).

### Phase 7 – Post-Migration Optimisation
23. Tune Analytics-tier vs. Data-Lake-tier placement based on actual query patterns and cost.
24. Review and retire any interim dual-forwarding/bridging setup used during migration.
25. Run a lessons-learned review and update the HLD to "as-built" status.
