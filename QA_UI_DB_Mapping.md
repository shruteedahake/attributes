# QA Reference: UI Field ↔ Postgres DB Mapping

**Purpose:** cross-verify what's displayed on screen against the exact database table/column it comes from, across all 3 personas. Built by tracing every page's `fetch()` calls through to the Vite-plugin route handler that runs the SQL (`claims-solution-integration/vite-plugins/*.ts`), plus the Python MCP backends where a feature calls out to those instead.

**How to read the tables:** `DB Table` / `DB Column(s)` is the actual source of truth for verification. `Notes` flags anything non-obvious — derived/computed values, fields that are 100% mock (don't bother checking the DB for these), or fields populated by a different backend process than this app's own Vite plugins.

---

## Table of Contents
1. [Policyholder Persona](#policyholder-persona)
2. [Adjuster Persona](#adjuster-persona)
3. [SIU Persona](#siu-persona)
4. [Cross-Cutting Gotchas](#cross-cutting-gotchas)

---

## Policyholder Persona

**Pages covered:** My Claims, Follow My Claims, Document Hub, Smart Loss Reporting (FNOL wizard).
**Backend traced:** `vite-plugins/claims.ts`, `claim-journey.ts`, `policy-details.ts`, `documents.ts`, `fnol-submission.ts`, `voice-extraction.ts`, `claim-insights.ts`, `db.ts`.

| Page | Section | UI Field Label | DB Table | DB Column(s) | Notes |
|---|---|---|---|---|---|
| My Claims | Claim Card | Claim ID (top chip) | claims | claim_number | `/api/claims` (claims.ts) |
| My Claims | Claim Card | Status pill | claims | status | |
| My Claims | Claim Card | Title/description | claims | short_description | |
| My Claims | Claim Card | Date (calendar icon) | claims | filed_at | Formatted via `formatDate()` |
| My Claims | Claim Card | Type (file icon) | claims | loss_type | Also drives left-border color |
| My Claims | Claim Card | Location (pin icon) | claims | location | "—" if null |
| My Claims — Summary Modal | Claim Information | Claim Number | claims | claim_number | |
| My Claims — Summary Modal | Claim Information | Policyholder | claims | policyholder_name | |
| My Claims — Summary Modal | Claim Information | Policy Number | claims | policy_number | |
| My Claims — Summary Modal | Claim Information | Date Filed | claims | filed_at | formatted |
| My Claims — Summary Modal | Claim Details | Loss Type | claims | loss_type | |
| My Claims — Summary Modal | Claim Details | Date of Loss | claims | date_of_loss | |
| My Claims — Summary Modal | Claim Details | Property / Location | claims | location | |
| My Claims — Summary Modal | Claim Details | Estimated Cost | claims | estimated_cost | |
| My Claims — Summary Modal | Claim Details | Severity | claims | severity | |
| My Claims — Summary Modal | Claim Details | Coverage badge | claims | coverage | DERIVED: `true→"Covered"`, else `"Under Review"` |
| My Claims — Summary Modal | Claim Details | Short Description (under Coverage) | claims | ai_generated_summary (fallback short_description) | DERIVED fallback chain |
| My Claims — Summary Modal | Assessment Summary | Summary paragraph | claims | ai_generated_summary (fallback short_description) | Same derived value reused |
| My Claims — Summary Modal | Assessment Summary | AI Confidence | claims | ai_confidence | Null → "Not scored" |
| Follow My Claims | Claim Selector List | Claim entries | claims | claim_number, loss_type, status | Same `/api/claims` call |
| Follow My Claims | Journey Stepper | Stage labels (8 fixed names) | — | — | MOCK — hardcoded `JOURNEY_STAGES` array |
| Follow My Claims | Journey Stepper | Which stage is "current" | claim_journey_master | current_stage | DERIVED: `clamp(current_stage-1, 0, 7)` |
| Follow My Claims | Journey Stepper | Stage detail text on click | claim_journey_master | sub_status, current_stage_name | DERIVED sentence |
| Follow My Claims | Journey Stepper | "Routed to: X" | segmentation_result_output / stp_classification | stp_category, complexity | Via `/api/claim-insights` |
| Follow My Claims | Header Bar | CLAIM ID / POLICY NUMBER / POLICYHOLDER / LOSS TYPE / DATE OF LOSS | claims | claim_number, policy_number, policyholder_name, loss_type, date_of_loss | |
| Follow My Claims | Header Bar | SEVERITY (if present) | segmentation_result_output | severity | Via `/api/claim-insights` |
| Follow My Claims | Header Bar | "Stage N: name" badge | claim_journey_master | current_stage, current_stage_name | Falls back to hardcoded stage list if null |
| Follow My Claims | Header Bar | Status badge | claims | status | |
| Follow My Claims | Header Bar | STP category badge | stp_classification | stp_category | Via `/api/claim-insights` |
| Follow My Claims | Claim Intelligence: Readiness | Overall result badge | intake_validation_result_output | overall_result | |
| Follow My Claims | Claim Intelligence: Readiness | Completeness % + bar | intake_validation_result_output | completeness_score | |
| Follow My Claims | Claim Intelligence: Readiness | Missing fields list | intake_validation_result_output | missing_fields | Parsed array; labels via hardcoded `FIELD_LABELS` dict |
| Follow My Claims | Claim Intelligence: Readiness | "No documents uploaded yet" banner | intake_validation_result_output | docs_status | Shown when `docs_status === "Incomplete"` |
| Follow My Claims | Claim Intelligence: Processing Track | STP category + description | stp_classification | stp_category | Description text hardcoded per category value |
| Follow My Claims | Claim Intelligence: Processing Track | Complexity | segmentation_result_output | complexity | |
| Follow My Claims | Claim Intelligence: Coverage Status | Coverage verdict | coverage_verification_results | coverage_verdict | |
| Follow My Claims | Claim Intelligence: Coverage Status | Est. payable | coverage_verification_results | net_payable | DERIVED display logic |
| Follow My Claims | Claim Intelligence: Coverage Status | Exclusion note | coverage_verification_results | exclusion_triggered, exclusion_details | Only if exclusion_triggered true |
| Follow My Claims | What's Happening Now | Body text | claim_journey_master | sub_status, current_stage_name | DERIVED sentence |
| Follow My Claims | What Happens Next | Next stage name | — | — | MOCK — hardcoded JOURNEY_STAGES array |
| Follow My Claims | What Happens Next | Status chip | claim_journey_master | overall_sla_status (fallback sub_status) | `prettySla()` reformats |
| Follow My Claims | How Was This Stage? (feedback) | Clear/Confusing/Took Too Long buttons | — | — | Writes via Python MCP (feedback endpoint) — likely `customer_feedback_per_stage`, **not verified against this table's exact schema**, external to vite-plugins |
| Follow My Claims | Raise a Concern | Concern type + submit | — | — | Writes via Python MCP (communication log endpoint) — likely `communication_history`, **not verified here** |
| Follow My Claims | Latest Updates | Update title / Actor / Detail / Timestamp | communication_history | subject, handled_by, summary, communication_date | Summary truncated to 220 chars client-side |
| Follow My Claims | Estimated Timeline | Progress % + bar | claim_journey_master | current_stage | DERIVED: `round(current_stage/8*100)` |
| Follow My Claims | Estimated Timeline | Est. completion | claim_journey_master | expected_completion_date | Fallback "To be determined" |
| Follow My Claims | Document Share | 3 document-type rows | — | — | MOCK — hardcoded labels, just deep-links to Document Hub |
| Follow My Claims | Contact Team | Email Adjuster / Escalate | — | — | Writes via Python MCP `log_inbound` — likely `communication_history`, external to vite-plugins |
| Follow My Claims | Contact Team | Download Claim Summary | (reuses all already-fetched fields) | — | Client-side text file assembly, no new DB call |
| Document Hub | Claim Selector | Claim ID + loss type + doc count | claims, documents | claim_number, loss_type, COUNT(documents.*) | Filtered by `uploaded_by_role='Policyholder'` for this persona |
| Document Hub | Header chip | "claimId · policyholderName" | claims | claim_number, policyholder_name | |
| Document Hub | Header chip | "N documents on record" | documents | COUNT(*) | |
| Document Hub | Filters sidebar | Photos/Reports/Estimates/Invoices counts | documents | document_type, content_type, file_name | DERIVED via `categorize()` |
| Document Hub | Document row | File name / Category / Size / Uploaded date / Status | documents | file_name, document_type, file_size, uploaded_at, status | Size = `round(file_size/1024)` KB; null status shown as "Pending" |
| Document Hub | Timeline view | Role dot/badge | documents | uploaded_by_role | Colors via hardcoded maps keyed on value |
| Document Hub | Timeline view | Grouped date headers | documents | uploaded_at | DERIVED grouping |
| Document Hub | AI Insights panel | Name/Classification/Confidence/Size/Uploaded/Role/Status | documents | file_name, document_type, classification_confidence, file_size, uploaded_at, uploaded_by_role, status | |
| Document Hub | AI Insights panel | "AI Generated" badge + confidence | documents | investigation_notes | DERIVED regex-parse of an embedded "Authenticity Verdict: …" string |
| Document Hub | AI Insights panel | AI Summary text | documents | insights | DERIVED JSON-parse for `summary`/`confidence_score` |
| Document Hub | AI Insights panel | Extracted Data rows | documents | extracted_data | DERIVED text parsing into label/value rows |
| Document Hub | AI Insights panel | "View original file" | documents | file_url, file_name, content_type | Streamed via `/api/document-file` proxy |
| Document Hub | Footer | "Access Rules" note | — | — | MOCK — static text |
| Smart Loss Reporting — Step 1 | Policy Number input + lookup result | Policy Number, verdict | — (writes `policy_details` on save) | policy_number | Lookup hits Python MCP `gw_search_policy` (Guidewire), **not local Postgres read**; a found policy is persisted via `save_policy_details` |
| Smart Loss Reporting — Step 1 | Describe | Description textarea | fnol_voice_text_extraction (voice only) | transcribed_text (fallback raw_input) | Typed text bypasses DB; voice path re-fetches from this table |
| Smart Loss Reporting — Step 1 | Describe | Estimated Damage Amount | fnol_submissions (persisted at Step 3) | estimated_cost | Later copied to `claims.estimated_cost` by backend submit flow |
| Smart Loss Reporting — Step 2 | Chat transcript | Message text | — | — | Live SSE chat with FNOL agent; its own tables (`fnol_ai_inferences`, `fnol_mandatory_question_log`) are outside this Vite app, not verified here |
| Smart Loss Reporting — Step 2 | Add Evidence | Attached files | documents (post-upload) | file_name | Uploaded via Python MCP `document_submission`; not read back on this step |
| Smart Loss Reporting — Step 2 | Your Policy Information | Policy Number / Insured Name / Insured Address / Policy Period | policy_details | policy_number, policyholder_name, policy_address (fallback city/state/postal_code/country), effective_date + expiration_date | `/api/policy-details`; matched via policy_number OR gw_policy_id OR account_number |
| Smart Loss Reporting — Step 2 | AI Extraction table (Field col) | Type of Loss, Cause of Loss, Area Affected, Date of Loss, Time of Loss, Sudden vs Gradual, Occupancy, Severity, Emotional Context, Urgency Indicator, Est. Damage Amount | fnol_submissions | loss_type, cause_of_loss, area_affected, date_of_loss, time_of_loss, sudden_vs_gradual, occupancy_at_loss, severity, emotional_context, urgency_indicator, estimated_cost | `/api/fnol-submission`; one row per `FIELD_DEFS` mapping |
| Smart Loss Reporting — Step 2 | AI Extraction table (Source col) | AI Extracted / From Your Voice / From Description / Confirmed by You / You Provided / Policy Record / Human Edited | fnol_submissions | `<field>_source` columns | DERIVED text via `mapSource()` lookup |
| Smart Loss Reporting — Step 2 | AI Extraction table (Human Review col) | Editable value per field | fnol_submissions | same value columns | PATCH updates value column + sets `_source` to `human_edited` |
| Smart Loss Reporting — Step 2 | — | Overall Agent Confidence % | fnol_submissions | overall_confidence | |
| Smart Loss Reporting — Step 3 | Review Form | Policy Number (top) | — | — | Client-side state, not re-queried |
| Smart Loss Reporting — Step 3 | Review Form | Attached Evidence list | — | — | Client-staged files, not yet `documents` rows |
| Smart Loss Reporting — Step 3 | Section A: Policy & Insured | Policy Number / Insured Name / Insured Address | policy_details | policy_number, policyholder_name, policy_address (fallback concat) | Re-fetched from `/api/policy-details` |
| Smart Loss Reporting — Step 3 | Section A | "Auto-Filled" source tag | — | — | MOCK — hardcoded label |
| Smart Loss Reporting — Step 3 | Section B: Loss Details | Same 11 fields as Step 2 | fnol_submissions | same columns as Step 2 | Reuses fetched state; "Edited"/"Human Edited" shown client-side if changed |

---

## Adjuster Persona

**Pages covered:** Loss Dashboard, Loss Investigation (Queue), Case Investigation, Expert Dispatch, Loss Assessment, Repair vs Replacement, Smart Vendor Match, Verification Intelligence.
**Backend traced:** `vite-plugins/adjuster.ts` (primary — handlers for each page), `claims.ts`, `documents.ts`. **Important:** several Loss Assessment panels and the HITL approvals banner call the **OrchestratorAgent's own server/MCP** (`ADJUSTER_ORCHESTRATOR_URL` / `ADJUSTER_ORCHESTRATION_MCP_URL`) — a separate backend process from this Vite app's own plugins. Flagged per-row below.

| Page | Section | UI Field Label | DB Table | DB Column(s) | Notes |
|---|---|---|---|---|---|
| Loss Dashboard | KPI Tiles | Assigned Claims | claims | (COUNT all rows) | Derived count |
| Loss Dashboard | KPI Tiles | Avg Resolution | claim_journey_master | total_days_in_journey | Derived average (>0 only) |
| Loss Dashboard | KPI Tiles | Claims at Risk | claims | severity | Count where high/critical |
| Loss Dashboard | KPI Tiles | Approved Today | claims | status, filed_at | Derived count, filed within 1 day |
| Loss Dashboard | Avg STP Readiness band | Readiness % + Eligibility label | stp_classification | stp_category (latest per claim) | Derived % + label |
| Loss Dashboard | STP category cards | Full STP / Vendor-STP / Fast Track / Manual counts | stp_classification | stp_category | Bucketed by substring match |
| Loss Dashboard | Priority Claims | Claim # / Description / Loss Type / Location / Severity | claims | claim_number, short_description (fallback detected_cause), loss_type, location, severity | Filtered severity high/critical, first 6 |
| Loss Dashboard | Recent Activity | Policyholder / Description / Stage pill / (sort) Filed At | claims + claim_journey_master | policyholder_name, short_description, current_stage_name (fallback claims.status), filed_at | First 8 by filed_at DESC |
| Loss Dashboard | Pre-Loss Risk Alerts | Title / Description / Type / Severity / Location / Date | pre_loss_alerts | title, description, alert_type, severity, location, alert_date | `WHERE resolved IS NOT TRUE ORDER BY alert_date DESC LIMIT 6` |
| Loss Investigation (Queue) | Banner chips | Active Cases | claims | (row count) | |
| Loss Investigation (Queue) | Banner chips | Pending Docs | documents | flagged | Count where flagged ≠ 0 |
| Loss Investigation (Queue) | Banner chips | SIU Reviews | claim_triage | routing | Count containing "siu" |
| Loss Investigation (Queue) | Table | Claim # / Policyholder / Loss Type / Status / Filed | claims | claim_number, policyholder_name, loss_type, status, filed_at | |
| Loss Investigation (Queue) | Table | Severity / Complexity | claims (fallback claim_triage) | severity (fallback damage_severity), complexity (fallback complexity) | |
| Loss Investigation (Queue) | Table | Approval Mode | stp_classification | stp_category (latest) | |
| Case Investigation | Header card | Claim # / Policyholder / Loss Type / Location / Severity / Status | claims | claim_number, policyholder_name, loss_type, location, severity, status | |
| Case Investigation | Pending HITL Approvals banner | Gate label / Summary | human_approval_requests | gate_type, summary | **Cross-backend** — read via OrchestratorAgent's own MCP, not vite-plugins |
| Case Investigation | Evidence Review Panel | Uploaded Evidence Photos | documents | document_id, file_name, uploaded_at | `content_type LIKE 'image/%' LIMIT 12`; served via `/api/document-file` |
| Case Investigation | AI Investigation Tools | Fraud Screening → score | ai_fraud_signals | fraud_score | Latest row |
| Case Investigation | AI Investigation Tools | Damage Assessment → severity | damage_items | severity | Latest row (keyed by claim_number despite column name) |
| Case Investigation | AI Investigation Tools | External Data → storm event | weather_location_alignment | storm_event | Latest row |
| Case Investigation | AI Investigation Tools | Verification → Confirmed/Flagged | verification_details + external_verifications | flag (severity='Critical') | DERIVED verdict |
| Case Investigation | AI Investigation Tools | Claim Classification → complexity | claim_triage | complexity | Latest row |
| Case Investigation | AI Investigation Tools | Evidence Validation → summary | documents | flagged, investigation_notes | Derived counts |
| Case Investigation | AI Investigation Tools | Triage → Priority Score | claim_triage | fraud_risk_score | Latest row |
| Case Investigation | AI Investigation Tools | Routing Tool | claim_triage | routing | Latest row |
| Case Investigation | AI Investigation Tools | Loss Assessment → net payable | loss_estimation_outputs | net_payable | Latest row |
| Case Investigation | AI Investigation Tools | Repair vs Replace → recommendation/costs | estimates | recommendation, repair_cost, replacement_cost | ⚠️ `estimates.claim_id` stores the **numeric `claims.id`**, not the claim_number string, unlike every other table on this page |
| Case Investigation | Intake Validation | Status / Completeness / Failure Reasons / Overridden By/Notes | claim_intake_validation | validation_passed, overridden, data_completeness_score, failure_reasons, overridden_by, overridden_notes | |
| Case Investigation | Investigation Completion | Completed / By / At | claim_investigation_completion | completed_by, completed_at | Row presence = completed |
| Case Investigation | Smart Comparison tiles | Similar Claims / Avg Approval Rate / Avg Settlement / Avg Resolution | claims (all rows) + ai_decision_recommendations + claim_journey_master | claim_number, loss_type, severity, estimated_cost, status, settlement_amount, total_days_in_journey | Heuristic similarity scoring, fully derived |
| Case Investigation | Similar Claims table | Claim # / Match Score / Amount / Settlement / Outcome / Days / Fraud Risk / Matching Factors | claims + ai_decision_recommendations + claim_journey_master + fraud_risk_snapshots | claim_number, estimated_cost, settlement_amount, total_days_in_journey, fraud_score | Match Score and Matching Factors chips are **fully derived heuristics, not stored columns** |
| Case Investigation | Adjuster Actions | Complete Claim Intake Validation (+override) | claim_intake_validation | overridden, overridden_by, overridden_notes | Also best-effort updates `claim_journey_master.sub_status` |
| Case Investigation | Adjuster Actions | Complete Investigation | claim_investigation_completion | claim_id, completed_by, completed_at | Insert on click |
| Case Investigation | Adjuster Actions | Request Additional Proof modal | communication_history | communication_id, claim_row_id, claim_number, policyholder_name, communication_type, direction, subject, summary, handled_by, resolution_status, follow_up_required | Also visible on Policyholder's Follow My Claims |
| Case Investigation | Adjuster Actions | Schedule Interview modal | communication_history | same columns (subject="Interview Scheduled") | |
| Expert Dispatch | Stat cards | Pending Dispatches | work_orders | status | Count in ('pending','scheduled') |
| Expert Dispatch | Stat cards | Overdue Visits | claim_journey_master + claims | overall_sla_status, filed_at | Derived breach flag |
| Expert Dispatch | Stat cards | Drone Combo Requests | drone_evidence_summary | claim_id (distinct) | |
| Expert Dispatch | Dispatch Map | Pin positions | — | — | MOCK — algorithmic layout, not geocoded |
| Expert Dispatch | Dispatch Queue | Claim # / Severity / Policyholder / Location / Loss Type | claims | claim_number, severity, policyholder_name, location, loss_type | |
| Expert Dispatch | Dispatch Queue | SLA Breached flag | claim_journey_master + claims | overall_sla_status, filed_at | Same derivation as stat card |
| Expert Dispatch | Dispatch Queue | "Recommended: X" specialty chip | — | — | Derived via code lookup table, not a DB column |
| Expert Dispatch | Dispatch Queue | Assign Expert / Virtual First / Drone Sweep / No Visit Needed buttons | — | — | **MOCK — no DB write, toast only** |
| Loss Assessment | Claim Selector | Search/Select / Customer Name / Policy Type / Status | claims + policy_details | claim_number, policyholder_name, status, coverage_type | Policy Type falls back to "Home" if no policy row |
| Loss Assessment | Damage Snapshot | Damage ID/Category/Severity/Cost/Notes | damage_items | damage_id, category, severity, estimated_cost, adjuster_notes | |
| Loss Assessment | Damage Snapshot | "Add New Damage" button | — | — | **MOCK — toast only** |
| Loss Assessment | Loss Calculation Engine | Parts/Labor Cost, Depreciation %, Deductible, Subrogation (editable) | loss_assessments | total_parts_cost, total_labor_cost, depreciation_percent, deductible, subrogation_likelihood | Editable, saved on Save |
| Loss Assessment | Loss Calculation Engine | Gross/Depreciation/Net Loss/Total Estimate | — | — | **DERIVED entirely client-side** from the 4 inputs above |
| Loss Assessment | Loss Calculation Engine | System Recommendation / Confidence % | repair_vs_replacement_decisions (fallback loss_assessments) | recommended_action (fallback system_recommendation), confidence_score | |
| Loss Assessment | Adjuster Decision Panel | Adjuster Override / Final Recommendation / Notes | loss_assessments + repair_vs_replacement_decisions | adjuster_override, notes, decision (fallback final_recommendation) | Saved via `/save-loss-assessment` |
| Loss Assessment | "Run AI Analysis" | — | — | — | Triggers OrchestratorAgent workflow continuation (**cross-backend**), then reloads this page's own GET |
| Loss Assessment | AI Claim Analysis Summary | Estimated Loss / Net Payable | loss_estimation_outputs | ai_estimated_loss, net_payable | |
| Loss Assessment | AI Claim Analysis Summary | Deductible Applied | loss_assessments | deductible | |
| Loss Assessment | AI Claim Analysis Summary | Reserve Recommendation | adjuster_findings | system_recommended_reserve | |
| Loss Assessment | AI Claim Analysis Summary | Repair vs Replace chip | repair_vs_replacement_decisions | recommended_action | |
| Loss Assessment | AI Claim Analysis Summary | STP Readiness Score + Category | ai_decision_recommendations (fallback segmentation_result_output) + stp_classification | stp_score, stp_category | |
| Loss Assessment | AI Claim Analysis Summary | Fraud Risk Level + Score | fraud_risk_snapshots | fraud_score | |
| Loss Assessment | AI Claim Analysis Summary | Subrogation Risk / Complexity / Severity | loss_assessments, claims | subrogation_likelihood, complexity, severity | |
| Loss Assessment | AI Claim Analysis Summary | Eligibility Rules chips | (multiple above) + adjuster_findings.coverage_confirmed | — | Each `pass` boolean fully derived |
| Loss Assessment | AI Recommendation panel | Recommendation text + Confidence bar | ai_decision_recommendations (fallback loss_estimation_outputs) | recommended_action, confidence | |
| Loss Assessment | Reserve Recommendation | System/Adjusted Reserve | adjuster_findings | system_recommended_reserve, adjusted_reserve | System value refreshed by **OrchestratorAgent** `/reserve-analysis` (cross-backend) |
| Loss Assessment | Reserve Recommendation | Variance / Buffer %s / Rationale | — | — | **Transient**, computed live by OrchestratorAgent — not persisted anywhere |
| Loss Assessment | Reserve Recommendation | Approve/Reject + Save Reserve | adjuster_findings | adjusted_reserve | Also best-effort decides `reserve_approval` HITL gate |
| Loss Assessment | Settlement Recommendation | Settlement Amount / Final Settlement | ai_decision_recommendations | settlement_amount, final_settlement_amount | |
| Loss Assessment | Settlement Recommendation | Deductible / Remaining Coverage Limit | loss_assessments, ai_decision_recommendations | deductible, remaining_coverage_limit | |
| Loss Assessment | Settlement Recommendation | STP Score/Action/Notes | — | — | **Transient** — OrchestratorAgent `/settlement-analysis`, not stored beyond final_settlement_amount |
| Loss Assessment | Settlement Recommendation | Approve/Reject + Save Settlement | ai_decision_recommendations | final_settlement_amount | Best-effort decides `settlement_approval` gate |
| Loss Assessment | Financial Leakage | Risk / Score / Variance / Costs / Flags / Recommendation | financial_leakage_score | leakage_risk, leakage_score, overall_variance_percent, total_estimated_cost, total_actual_cost, risk_flags, recommendation | |
| Loss Assessment | Financial Leakage | Risk Level Override / Adjuster Notes | financial_leakage_score | adjuster_override_risk_level, adjuster_notes | Best-effort decides `financial_leakage_review` gate |
| Loss Assessment | Payment Eligibility | Eligible/Decision/STP Category/Gates | — | — | **Transient** — OrchestratorAgent `/financial-leakage-analysis` eligibility block, not persisted |
| Loss Assessment | Payment Readiness & Final Decision | Approved for Payment / Amount / Reason | — | — | **Transient preview**, same call |
| Loss Assessment | Payment Readiness & Final Decision | Approve & Disburse / Reject | payment_disbursements | payment_id, amount | Via OrchestratorAgent `/payment-decision` (**cross-backend**); best-effort decides `payment_approval` gate |
| Loss Assessment | Claim Summary sidebar | Claim Number / Loss Date / Coverage Limit / Deductible / Status | claims + policy_details + loss_assessments | claim_number, date_of_loss (fallback filed_at), coverage_limit, deductible, status | |
| Loss Assessment | Claim Summary sidebar | "Recent Interactions" | — | — | **MOCK** — always "No recent interactions" |
| Repair vs Replacement | Claim Selector | Loss Type/Status/Severity/Fraud Score chips | claims + fraud_risk_snapshots | loss_type, status, severity, fraud_score | |
| Repair vs Replacement | Financial Impact Comparison | Total Cost (Repair/Replacement) | repair_costs, replacement_costs | material_cost, labor_hours, labor_rate, diagnostic_fee, total_repair_estimate / replacement_material_cost, installation_hours, delivery_fee, disposal_fee, total_replacement_estimate | Both queried via `item_id LIKE 'DMG-<claim>-%'` |
| Repair vs Replacement | Financial Impact Comparison | Net Payable (Repair) | loss_estimation_outputs | net_payable | Replacement column shows a derived total instead |
| Repair vs Replacement | Financial Impact Comparison | Time (days) | repair_costs / replacement_costs | labor_hours / installation_hours | Derived day counts |
| Repair vs Replacement | Financial Impact Comparison | AI Recommendation + Confidence | loss_estimation_outputs | repair_recommended, confidence | |
| Repair vs Replacement | Repair Scenario card | Cost/Material/Labor/Time | repair_costs | (sums, see above) | |
| Repair vs Replacement | Repair Scenario card | Vendor Availability / Downtime Impact | — | — | **MOCK — not backed by DB** |
| Repair vs Replacement | Replacement Scenario card | Cost/Material/Install/Lead Time | replacement_costs | (sums, see above) | |
| Repair vs Replacement | Replacement Scenario card | Depreciation Impact / Procurement Availability | — | — | **MOCK — not backed by DB** |
| Repair vs Replacement | AI Explainability Panel | Cost Threshold Comparison | (derived) | — | Not stored |
| Repair vs Replacement | AI Explainability Panel | Asset Age/Repair Feasibility/Turnaround factors | — | — | **MOCK** static text (Turnaround partially uses derived days) |
| Repair vs Replacement | AI Explainability Panel | Policy & Coverage Conditions | loss_estimation_outputs | net_payable | |
| Repair vs Replacement | AI Explainability Panel | Vendor Input & Historical Data | vendor_benchmarks | vendor_name, avg_repair_cost, avg_replacement_cost, eta_days | Matched by specialty |
| Repair vs Replacement | AI Explainability Panel | Subrogation Potential | fraud_risk_snapshots | fraud_score | |
| Repair vs Replacement | AI Explainability Panel | Decision Confidence footer | loss_estimation_outputs | confidence | |
| Smart Vendor Match | Banner | VIS Records / Vendors count | vendors | (row count) | |
| Smart Vendor Match | AI Recommendation | Required Specialty | — | — | Derived via code lookup table |
| Smart Vendor Match | AI Recommendation | Subrogation Potential | fraud_risk_snapshots | fraud_score | |
| Smart Vendor Match | Recommended Vendor card | Name/Rating/Avg Cost/ETA | vendors | name, rating, avg_cost, avg_turnaround_days | |
| Smart Vendor Match | Recommended Vendor card | VIS score badge | — | — | **DERIVED** multi-factor score, not stored |
| Smart Vendor Match | Recommended Vendor card | "SLA: On Track" / "Cost: +6%" / "Subro: Compliant" chips | — | — | **MOCK — static, always the same** |
| Smart Vendor Match | Recommended Vendor card | "Risk: X" chip | vendor_benchmarks | fraud_score | |
| Smart Vendor Match | Stat cards | License Verified / Avg Rating / Avg Turnaround / Total Jobs / STP Ready | vendors | license_valid, rating, avg_turnaround_days, completed_jobs, verified | All derived counts/averages |
| Smart Vendor Match | Ranked Vendor List | Name/Phone/STP badge/Specialty/License/Rating/Avg Cost/ETA/Jobs/Location | vendors | name, phone, verified, specialty, license_valid, license_number, rating, avg_cost, avg_turnaround_days, completed_jobs, city, state | |
| Smart Vendor Match | Ranked Vendor List | "SLA: On Track" chip | — | — | **MOCK** |
| Smart Vendor Match | Ranked Vendor List | "Risk: X" chip / VIS Score | vendor_benchmarks | fraud_score | VIS score itself derived |
| Smart Vendor Match | Ranked Vendor List | "Assign" button | — | — | **MOCK — toast only** |
| Verification Intelligence | Claim Selection | Insured Name/Loss Type/Location | claims | policyholder_name, loss_type, location | |
| Verification Intelligence | Claim Selection | Fraud Risk Score chip | fraud_risk_snapshots | fraud_score, red_flag_count | |
| Verification Intelligence | Customer Uploaded Images | Thumbnails + Uploaded date | documents | document_id, file_name, uploaded_at, uploaded_by_role, content_type | |
| Verification Intelligence | Drone Aerial Imagery | Captured time/Site Rating/Damage Match/Alignment | drone_evidence_summary | drone_capture_time, roof_condition_rating, damage_match_percent, weather_event_alignment | |
| Verification Intelligence | Drone Aerial Imagery | Site Condition (bottom box) | drone_authenticity_data | roof_condition | |
| Verification Intelligence | No Drone Data alert | "Request Drone Sweep" | — | — | Sends chat message to OrchestratorAgent (**cross-backend**); no direct DB write |
| Verification Intelligence | Adjuster Actions | Approve Verified / Forward to SIU | human_approval_requests | gate_type='damage_assessment_review', decision_notes | Via OrchestratorAgent MCP (**cross-backend**) |
| Verification Intelligence | Adjuster Actions | Request Clarification / Generate Report | — | — | **MOCK — toast only** |

**⚠️ Non-obvious key convention:** `estimates.claim_id` stores the internal numeric `claims.id`, while virtually every other Adjuster table's claim key stores the business claim number string (e.g. "CLM-2026-7002"). Convert via `claims.id` when checking `estimates` rows.

---

## SIU Persona

**Pages covered:** Fraud Dashboard, SIU Investigation Workbench (queue + case detail), Vendor Fraud Check, Document Hub (shared with other personas — see Policyholder section above).
**Backend traced:** `vite-plugins/siu.ts`, `vite-plugins/vendor-fraud.ts`. Everything here was built/verified directly in this session against the live database.

| Page | Section | UI Field Label | DB Table | DB Column(s) | Notes |
|---|---|---|---|---|---|
| Fraud Dashboard | Banner / KPI Tiles | Suspected Claims / Anomalies Flagged / Investigation Queue / Resolved This Week | — | — | **MOCK — 100% hardcoded, entire page has zero backend calls** |
| Fraud Dashboard | Flagged Claims | ID/Description/Type/Address/Severity | — | — | **MOCK — hardcoded array** |
| Fraud Dashboard | Recent Activity | Name/Description/Status | — | — | **MOCK — hardcoded array** |
| SIU Investigation Workbench — Queue | Banner | Open Cases / Escalated / Total Cases | siu_claim_master (via list) | status, — | DERIVED counts from the fetched case list, not a stored column |
| SIU Investigation Workbench — Queue | Case card | SIU Case ID | siu_case_master | siu_case_id | Shows `PENDING-<claimId>` if no case row yet |
| SIU Investigation Workbench — Queue | Case card | Claim ID | siu_claim_master | claim_id | Source-of-truth table for "which claims are in SIU" |
| SIU Investigation Workbench — Queue | Case card | Fraud badge | fraud_risk_snapshots | fraud_score | Latest row per claim |
| SIU Investigation Workbench — Queue | Case card | Policyholder | claims | policyholder_name | Joined via claim_number |
| SIU Investigation Workbench — Queue | Case card | Investigator | siu_case_master | assigned_investigator | |
| SIU Investigation Workbench — Queue | Case card | Status badge | siu_case_master (fallback siu_claim_master) | status | |
| SIU Case Detail | Case Header | Claim ID | siu_claim_master | claim_id | |
| SIU Case Detail | Case Header | SIU Case ID | siu_case_master | siu_case_id | |
| SIU Case Detail | Case Header | Policyholder | claims | policyholder_name | |
| SIU Case Detail | Case Header | Loss Type | siu_claim_master | loss_type | |
| SIU Case Detail | Case Header | Claim Amount | — | — | **MOCK — hardcoded "—", never wired** |
| SIU Case Detail | Case Header | Fraud Risk Score | fraud_risk_snapshots | fraud_score | Latest row |
| SIU Case Detail | Case Header | Referral Source | siu_escalation_records | escalated_by | Raw value shown verbatim |
| SIU Case Detail | Case Header | Assigned Investigator | siu_case_master | assigned_investigator | |
| SIU Case Detail | Fraud Risk Summary | Gauge score + risk badge | fraud_risk_snapshots | fraud_score | Same value as header; risk label derived client-side (≥70 High / ≥40 Medium / else Low) |
| SIU Case Detail | Fraud Risk Summary | Key Fraud Indicators (pattern + severity) | fraud_risk_flags_output | risk_flag, severity | All rows for the claim, written by FraudPatternAgent |
| SIU Case Detail | Summary (Entity Relationship Agent) | Suspicion Risk Score | fraud_network_graph | risk_score | Latest non-null row for the claim |
| SIU Case Detail | Summary (Network Analysis Agent) | Fraud Ring Detected (Yes/No) | siu_network_analysis_results | ring_detected | |
| SIU Case Detail | Summary (Behavioral Analytics Agent) | Anomaly Score | siu_behavioral_analysis | anomaly_score | Joined via `siu_case_master.siu_case_id` — this table has no claim_id column |
| SIU Case Detail | Summary (Behavioral Analytics Agent) | Tone Shift Detected | siu_behavioral_analysis | tone_shift_detected | |
| SIU Case Detail | Summary (Behavioral Analytics Agent) | Timing Anomaly Detected | siu_behavioral_analysis | timing_anomaly_detected | |
| SIU Case Detail | Summary (Evidence Correlation Agent) | Corroboration Score | siu_evidence_correlation_results | corroboration_score | |
| SIU Case Detail | Summary (Evidence Correlation Agent) | Overall Finding | siu_evidence_correlation_results | overall_finding | |
| SIU Case Detail | SIU Investigation Timeline — Timeline Structure | 6 stage icons (green=done / grey=pending) | siu_timeline_events | event_type (presence check) | Green iff a row exists for that stage's mapped event_type: Case Opened / SIU Review Initiated / Evidence Review Completed / Interview Scheduled / External Verification Triggered / Case Resolved |
| SIU Case Detail | SIU Investigation Timeline — Timeline View | Event rows + timestamp | siu_timeline_events | event_type, status, timestamp | Same 6-stage mapping; ungenerated stages show "Pending" |
| SIU Case Detail | Activities | Case Initiation / SIU Review Started / Evidence Review / Interview Stage / External Verification checklists | — | — | **MOCK — hardcoded, not backed by DB** |
| SIU Case Detail | Progress Tracking | Current Stage | siu_progress_tracker | stage | Latest row for the claim; written by SIUClosureAgent's `check_closure_readiness` |
| SIU Case Detail | Progress Tracking | Stage Progress % | siu_progress_tracker | progress_percent | |
| SIU Case Detail | Investigation Actions | Request Additional Proof / Documentation (text + Preview/Edit toggle) | — | — | **UI-only — not persisted anywhere.** "Send Request" just closes the panel |
| SIU Case Detail | Investigation Actions | Schedule Interview (mode/date/time/notes) | siu_timeline_events (on Send only) | event_type='Interview Scheduled', status='Completed' | Only the *fact* that an interview was scheduled is persisted — the chosen date/time/mode/notes are **not stored anywhere** |
| SIU Case Detail | Investigation Actions | Trigger External Verification (check selection) | siu_timeline_events (on Trigger only) | event_type='External Verification Triggered', status='Completed' | Only the fact of triggering is persisted — which checks were selected is **not stored** |
| SIU Case Detail | Investigation Actions | Investigation Notes textarea + Save Notes | — | — | **MOCK — disabled, not backed by DB** |
| SIU Case Detail | Fraud Decision | Confirm Fraud / Clear Claim (Not Fraud) / Inconclusive — Return to Adjuster buttons | siu_evidence_correlation_results | decision | Canonical values written: `"Fraud Confirmed"`, `"Claim Cleared (no fraud)"`, `"Return to Adjuster"` |
| SIU Case Detail | Fraud Decision | (same click) HITL gate | human_approval_requests | status, decision_notes, gate_type='siu_investigation_review' | Confirm Fraud + Clear Claim → status="Approved" (Phase 3 runs either way); Inconclusive → status="Rejected" (stops at Phase 2) |
| SIU Case Detail | Fraud Decision | "Decision Recorded" display | siu_evidence_correlation_results | decision | Re-fetched after the click |
| Vendor Fraud Check | KPI tile | Critical Alerts | vendor_red_flags | severity | Derived count where severity='Critical' |
| Vendor Fraud Check | KPI tile | High-Risk Vendors | — | — | **MOCK — hardcoded "5"** |
| Vendor Fraud Check | KPI tile | Escalated Cases | — | — | **MOCK — hardcoded "8"** + local per-click increment (not persisted) |
| Vendor Fraud Check | KPI tile | Network Signals | vendor_network_signals | (row count) | Derived count |
| Vendor Fraud Check — Vendor Risk Scores tab | Vendor list + detail panel | Name/Specialty/Score/Claims/Last Assessed/Drivers/AI Analysis | — | — | **MOCK — entire tab hardcoded, not backed by DB** |
| Vendor Fraud Check — Red Flag Alerts tab | Alert card | Title | vendor_red_flags | title (fallback alert_type) | |
| Vendor Fraud Check — Red Flag Alerts tab | Alert card | Description | vendor_red_flags | explanation | |
| Vendor Fraud Check — Red Flag Alerts tab | Alert card | Date | vendor_red_flags | created_at | |
| Vendor Fraud Check — Red Flag Alerts tab | Alert card | Alert Type | vendor_red_flags | alert_type | |
| Vendor Fraud Check — Red Flag Alerts tab | Alert card | Vendor | vendor_red_flags | vendor_id | |
| Vendor Fraud Check — Red Flag Alerts tab | Alert card | Related claims count | vendor_red_flags | related_claim_ids | Derived: length of the JSON array |
| Vendor Fraud Check — Red Flag Alerts tab | Alert card | Detection Logic | vendor_red_flags | triggering_logic | |
| Vendor Fraud Check — Red Flag Alerts tab | Alert card | Severity badge | vendor_red_flags | severity | |
| Vendor Fraud Check — Red Flag Alerts tab | Alert card | Reviewed/Escalated badges (initial) | vendor_red_flags | is_reviewed, is_escalated | Initial page-load state only |
| Vendor Fraud Check — Red Flag Alerts tab | Alert card | Reviewed by | vendor_red_flags | reviewed_by | |
| Vendor Fraud Check — Red Flag Alerts tab | Alert card | "Mark Reviewed" / "Escalate" buttons | — | — | **Local React state only — clicking does NOT write back to `vendor_red_flags`** |
| Vendor Fraud Check — Network Signals tab | Signal card | Title | vendor_network_signals | signal_type | |
| Vendor Fraud Check — Network Signals tab | Signal card | Severity | vendor_network_signals | severity | |
| Vendor Fraud Check — Network Signals tab | Signal card | Vendor | vendor_network_signals | vendor_id | |
| Vendor Fraud Check — Network Signals tab | Signal card | Related Entity / Entity Type | vendor_network_signals | related_entity, related_entity_type | |
| Vendor Fraud Check — Network Signals tab | Signal card | Occurrences | vendor_network_signals | occurrence_count | |
| Vendor Fraud Check — Network Signals tab | Signal card | Note | vendor_network_signals | risk_narrative | |
| Vendor Fraud Check — Invoice Patterns tab | Table rows | Claim/Policyholder/Loss/Amount/Ratio/Flags | — | — | **MOCK — entire tab hardcoded, not backed by DB** |

---

## Cross-Cutting Gotchas

These apply across personas — worth knowing before testers start filing "bug" tickets against expected behavior:

1. **Shared/duplicated tables with divergent shapes** — `CREATE TABLE IF NOT EXISTS` is a no-op once a table exists, so whichever persona's `init_db.py` ran *first* determines the live schema. Two pairs of same-named tables have genuinely different column shapes depending on which persona created them first:
   - `policy_details` — Policyholder's version is the rich canonical shape (policy_number, coverage_limit, exclusions, etc.); Adjuster's/SIU's own declaration is a much narrower shape (policy_id, "limit") that never actually takes effect once Policyholder's version exists.
   - `cost_variance_output` — Adjuster's version has `created_at`, no UNIQUE; VendorManager's has `vendor_id UNIQUE`, no `created_at`.
   - `segmentation_result_output` — Policyholder keys on `claim_number`; Adjuster's own declaration keys on `claim_id` instead — different key column entirely.
   - `intake_validation_result_output` — Policyholder's version is a rich shape (fraud_flags, missing_docs, overall_result); Adjuster's is explicitly a **different, narrower re-check signal** per its own code comment — not meant to be the same data even though the name matches.
2. **`repair_vs_replacement_decisions`** is written to and read from by live Adjuster code but has **no `CREATE TABLE` in any of the 5 personas' `init_db.py`** — it only exists because it was created out-of-band on the live DB at some point. If this table is ever missing on a fresh environment, those writes will silently fail with no schema to create it from.
3. **Cross-backend fields are common** — several "live" panels (Adjuster's Reserve/Settlement/Financial-Leakage/Payment/HITL-approvals, SIU's Fraud Decision → Phase 3) are populated by a *different backend process* than the page's own Vite-plugin API (the OrchestratorAgent's MCP for Adjuster, the SIUOrchestrator's MCP for SIU). Persisted results land in the tables noted in each row, but transient/computed display text (rationale strings, buffer percentages, gate pass/fail booleans) often exists only in that one response and isn't stored in any single column — don't go looking for those in the DB.
4. **A large fraction of "AI-feeling" chips and buttons are 100% mock** across all 3 personas — every row marked **MOCK** above has zero backend call. If a tester changes DB data and a MOCK field doesn't update, that's expected, not a bug.
5. **`estimates.claim_id`** (Adjuster) stores the internal numeric `claims.id`, not the business claim number string — the one exception to the claim-key convention used everywhere else.
