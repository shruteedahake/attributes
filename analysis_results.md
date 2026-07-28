# Gap Analysis & Technical Roadmap: Extending Commercial Property Claims Platform to Motor Insurance

This document provides a deep architectural review of the claims platform codebase. It identifies the commercial property assumptions embedded in the database schema, business rules, API integrations, and LLM prompts, and outlines the required changes to support both **Commercial Property** and **Motor Insurance** in parallel without breaking the existing implementation.

---

## 1. Executive Summary

The existing claims platform is designed as an agentic, multi-persona claims lifecycle system (Orchestrator, Policyholder, Adjuster, Special Investigation Unit (SIU), and Vendor Manager). However, its data models, calculations, prompts, and verification integrations are heavily coupled with Commercial Property claims (specifically homeowners, storm, flood, roof, and plumbing losses).

To support Motor Insurance claims concurrently:
- **Core Architecture**: We must transition from hardcoded assumptions to a **Line of Business (LOB)-driven Strategy Pattern** where the Orchestrator and individual agents load behavior, validation rules, schemas, and prompts dynamically based on the claim LOB (e.g., `Property` vs. `Motor`).
- **Data Model**: The database requires generalization. The core `claims` and `policy_details` tables should be LOB-agnostic, referencing LOB-specific extension tables (`vehicle_details`, `driver_details`, `accident_details`, `motor_policy_coverages`) for motor claims.
- **Agent Decision Logic**: Calculations for damage, depreciation, reserves, and total loss must be abstracted. Property calculations (e.g., building contractor labor rates, materials depreciation) must run in parallel with motor calculations (e.g., part-specific depreciation scales, constructive total loss thresholds based on Insured Declared Value (IDV), and RTO/license validations).

Implementing these recommendations will allow the platform to scale to Motor Insurance claims while maintaining 100% backward compatibility with the existing Commercial Property claims.

---

## 2. Agent-Wise Gap Analysis

The claims platform features 47+ agents across five personas. Below is a detailed gap analysis for each agent:

### 2.1. Orchestrator Persona

#### OrchestratorAgent (Brain)
- **Current Property Logic**: Coordinates a 10-stage lifecycle. It hardcodes property-specific steps like executing weather/drone checks in Stage 3 and routing structural assessments.
- **Motor Insurance Requirements**: Needs conditional execution branches. For Motor LOB, Stage 3 must run vehicle RTO/VIN validation and driver license checks instead of aerial weather/drone sweeps. Total loss checking must run before vendor assignment.
- **Reusable Logic**: The core state machine, human-in-the-loop (HITL) gate management, and the `human_approval_requests` tables remain unchanged.
- **Code Changes Required**:
  - Modify `server.py` to inspect the LOB of the claim at startup.
  - Dynamically load stage sub-flows (e.g., run `RTO/VIN` verification instead of `drone` verification in Stage 3).
- **Estimated Development Effort**: **Medium**

---

### 2.2. Policyholder Persona

#### 1. VoiceTextIntakeAgent
- **Current Property Logic**: Conversational Q&A is focused on property fields (affected rooms, property address, occupancy, sudden vs. gradual water leak).
- **Motor Insurance Requirements**: Must capture driver details, vehicle registration, VIN, collision type, passenger counts, towing needs, and police report availability.
- **Reusable Logic**: Conversation history rebuilding, LangGraph routing, and SSE streaming remain unchanged.
- **Code Changes Required**: Update prompts to dynamically pull mandatory fields from the database based on policy LOB.
- **Estimated Development Effort**: **Medium**

#### 2. DuplicateClaimCheckAgent
- **Current Property Logic**: Compares reported loss dates and property addresses to flag potential duplicate claims.
- **Motor Insurance Requirements**: Must match duplicate filings using vehicle VIN/registration number, driver license, and accident timestamps/locations.
- **Reusable Logic**: Basic comparison structure and matching algorithm.
- **Code Changes Required**: Update `check_duplicate` tool in `duplicate_check_router.py` to condition its query parameters on the claim's LOB.
- **Estimated Development Effort**: **Low**

#### 3. FNOLOrchestrator
- **Current Property Logic**: Directs the intake flow assuming 7 mandatory property fields.
- **Motor Insurance Requirements**: Must orchestrate the extraction of vehicle info and validate policy details using RTO/Guidewire clients.
- **Reusable Logic**: Background task execution, speech-to-text transcription utilities.
- **Code Changes Required**: Update `server.py` and `_run_background_processing` to load different validation suites depending on the LOB.
- **Estimated Development Effort**: **Medium**

#### 4. DocumentSubmissionAgent
- **Current Property Logic**: Classifies and processes plumber reports, building repair estimates, and home damage photographs.
- **Motor Insurance Requirements**: Must support driving licenses, vehicle registration certificates (RC Book), insurance certificates, garage repair estimates, and dashcam videos.
- **Reusable Logic**: Blob storage upload handlers, Azure Document Intelligence helper, base64 conversion.
- **Code Changes Required**:
  - Update `classify_document` tool prompt to include motor document types.
  - Register new parser models for RTO RC Books and Driving Licenses.
- **Estimated Development Effort**: **Medium**

#### 5. PolicyCoverageVerificationAgent
- **Current Property Logic**: Hardcoded checks for homeowners policy coverage (Section I limits, property deductible, building exclusions).
- **Motor Insurance Requirements**: Verify Own Damage (OD), Third Party Liability (TP), Zero Depreciation, and Add-ons (Engine Protection, Consumables). Validate driver license expiry.
- **Reusable Logic**: Guidewire API client integration shell, save/read local DB policy details.
- **Code Changes Required**: Rewrite `verify_coverage` tool inside `policy_coverage_mcp/handler.py` to evaluate motor-specific rules and exclusions.
- **Estimated Development Effort**: **High**

#### 6. ClaimReadinessAgent
- **Current Property Logic**: Checks if mandatory property fields (occupancy, address) and property documents (plumber invoice) are present.
- **Motor Insurance Requirements**: Check for vehicle details, driver details, DL copy, RC Book, police FIR, and damage photos.
- **Reusable Logic**: Readiness scoring framework, completeness score arithmetic.
- **Code Changes Required**: Modify `score_claim_readiness` tool to dynamically load LOB-driven checklist metadata.
- **Estimated Development Effort**: **Low**

#### 7. ClaimSegmentationAgent
- **Current Property Logic**: Evaluates straight-through processing (STP) based on simple water/storm categories and low estimate thresholds.
- **Motor Insurance Requirements**: Minor glass claims or minor bumper/scratch repairs should route to STP. Major collisions involving bodily injury or total loss must route to manual adjuster reviews.
- **Reusable Logic**: Scoring arithmetic, result persistence tables.
- **Code Changes Required**: Update `compute_stp_score` tool to incorporate vehicle impact parameters and third-party liabilities.
- **Estimated Development Effort**: **Medium**

#### 8. ClaimStatusAgent
- **Current Property Logic**: Standard property stages (Claim Initiated, Under Review, Under Investigation, Approved, Closed).
- **Motor Insurance Requirements**: Include motor stages (Surveyor Dispatched, Vehicle at Garage, Repair Approved, Repair Completed).
- **Reusable Logic**: Stage SLA tracking logs, history updates.
- **Code Changes Required**: Make stage-name mappings configurable in `claim_status_router.py`.
- **Estimated Development Effort**: **Low**

#### 9. CommunicationAgent
- **Current Property Logic**: General policyholder inquiry responses and notifications about adjuster visits.
- **Motor Insurance Requirements**: Notifications for towing dispatch, network garage appointments, and surveyor visits.
- **Reusable Logic**: Inbound log system, sentiment analysis.
- **Code Changes Required**: Update prompt library to include auto-insurance communication flows.
- **Estimated Development Effort**: **Low**

#### 10. FeedbackAgent
- **Current Property Logic**: Collects feedback and satisfaction scores on property claims.
- **Motor Insurance Requirements**: Tracks satisfaction across motor milestones (towing dispatch, cashless garage experience, repair quality).
- **Reusable Logic**: Rating writing logic, sentiment tracking.
- **Code Changes Required**: Modify schema references to support motor stages in feedback reports.
- **Estimated Development Effort**: **Low**

---

### 2.3. Adjuster Persona

#### 1. ClaimClassificationAgent
- **Current Property Logic**: Classifies losses as Water, Fire, Storm, Hail, Theft, Wind, Lightning, Snow, Ice.
- **Motor Insurance Requirements**: Classify into Collision, Theft, Fire, Flood, Vandalism, Hit and Run, Glass Damage, Animal Collision.
- **Reusable Logic**: Classification routing, LLM wrapper.
- **Code Changes Required**: Update the classification tool prompt in `claim_classification_mcp` to support motor classifications.
- **Estimated Development Effort**: **Low**

#### 2. TriageAgent
- **Current Property Logic**: Priority based on building damage severity, occupancy, and risk of secondary damage (mold, structural failure).
- **Motor Insurance Requirements**: Triage based on vehicle drivability, towing needs, bodily injury, passenger counts, and third-party liabilities.
- **Reusable Logic**: Triage scoring skeleton, adjuster workload lookup.
- **Code Changes Required**: Update the `run_triage` tool to use LOB-specific severity and complexity scoring rules.
- **Estimated Development Effort**: **Medium**

#### 3. FraudScreeningAgent
- **Current Property Logic**: Flags property fraud (manipulated drone imagery, weather history mismatch, contractor collusion).
- **Motor Insurance Requirements**: Flag staged accidents, VIN/RC mismatch, odometer tampering, prior damage, duplicate auto images, and telematics inconsistencies.
- **Reusable Logic**: Flag listing, risk snapshot recording.
- **Code Changes Required**: Integrate RTO verification and license validations as fraud screening rules.
- **Estimated Development Effort**: **High**

#### 4. RoutingAgent
- **Current Property Logic**: Routes property claims to general property adjusters, commercial property specialists, or catastrophe teams.
- **Motor Insurance Requirements**: Route to auto adjusters, bodily injury specialists, total loss experts, or third-party claim handlers.
- **Reusable Logic**: static assignment log lookup.
- **Code Changes Required**: Add specialized motor expertise queues to the routing algorithms.
- **Estimated Development Effort**: **Low**

#### 5. EvidenceValidationAgent
- **Current Property Logic**: Cross-validates property repair estimates and plumber diagnostics against the reported loss.
- **Motor Insurance Requirements**: Cross-validate garage estimates, surveyor reports, telematics logs, and police FIR details.
- **Reusable Logic**: Evidence status tracking database writes.
- **Code Changes Required**: Modify prompt to process vehicle-related evidence packages.
- **Estimated Development Effort**: **Medium**

#### 6. ExternalDataAgent
- **Current Property Logic**: Fetches historical weather data (Open-Meteo) and simulates aerial drone inspection summaries (roof condition).
- **Motor Insurance Requirements**: Integrate with RTO (registration checks), license validation APIs, vehicle history databases, telematics logs, and weather/map APIs.
- **Reusable Logic**: Historical weather lookup client.
- **Code Changes Required**:
  - Add RTO/VIN search and license checks.
  - Implement a simulated surveyor/vehicle inspection report tool.
- **Estimated Development Effort**: **High**

#### 7. DamageAssessmentAgent
- **Current Property Logic**: Sums damaged property items. Categorizes items into building components (Flooring, Drywall, Plumbing, HVAC) with hardcoded building contractor labor rates ($75/hr) and diagnostic fees ($150).
- **Motor Insurance Requirements**: Categorize damage by vehicle parts (fender, bumper, windshield, engine, ADAS sensors). Apply motor mechanic, body work, and paint labor rates.
- **Reusable Logic**: Vision/PDF document parsing helpers, image download helpers.
- **Code Changes Required**:
  - Remove hardcoded contractor formulas (`_compute_item_costs`).
  - Introduce dynamic parts catalogs and labor rate models based on repair shop networks.
- **Estimated Development Effort**: **High**

#### 8. VerificationAgent
- **Current Property Logic**: Verifies property ownership, loss dates against active coverage, and location details.
- **Motor Insurance Requirements**: Verify vehicle ownership (RC details), driver license class (e.g., commercial vs. private vehicle license), and geographical boundaries of the accident.
- **Reusable Logic**: Date-in-policy-window checking logic, verification logs.
- **Code Changes Required**: Update prompt to check driver and vehicle registry states.
- **Estimated Development Effort**: **Medium**

#### 9. LossAssessmentAgent
- **Current Property Logic**: Applies a flat 30% labor cost heuristic and property severity-based depreciation (5% to 30%).
- **Motor Insurance Requirements**: Calculate labor based on actual hours * hourly mechanic rate. Apply standard parts-based depreciation scales (plastic/rubber 50%, metal parts based on age).
- **Reusable Logic**: Deductible deduction, net payable calculation structure.
- **Code Changes Required**:
  - Rewrite `run_loss_assessment` to apply motor depreciation scales and actual labor costs.
  - Add zero-depreciation cover waiver checking logic.
- **Estimated Development Effort**: **High**

#### 10. ReserveRecommendationAgent
- **Current Property Logic**: Computes reserves based on building repair labor and material costs.
- **Motor Insurance Requirements**: Reserves must include parts, labor, towing, storage, third-party liability, and bodily injury medical reserves.
- **Reusable Logic**: Reserve logging database schema.
- **Code Changes Required**: Implement reserve estimation logic using garage cost benchmark databases.
- **Estimated Development Effort**: **Medium**

#### 11. FinancialLeakageAgent
- **Current Property Logic**: Checks variance between initial contractor estimates and final invoices.
- **Motor Insurance Requirements**: Check for duplicate parts, unauthorized OEM upgrades, inflated paint/labor hours, and invoice duplication.
- **Reusable Logic**: Variance math, database log structure.
- **Code Changes Required**: Modify leakage checking rules to flag auto repair discrepancies.
- **Estimated Development Effort**: **Medium**

#### 12. RepairVsReplacementAgent
- **Current Property Logic**: Recommends replacing if repair cost exceeds 60% of the replacement cost, or if age exceeds useful life.
- **Motor Insurance Requirements**: Constructive Total Loss (CTL) recommendation if repair estimate exceeds 75% of the vehicle's Insured Declared Value (IDV). Incorporate salvage value deductions.
- **Reusable Logic**: Cost estimation structure.
- **Code Changes Required**: Modify the comparison algorithm in `compare_repair_vs_replace` to use IDV and CTL criteria.
- **Estimated Development Effort**: **Medium**

#### 13. SettlementRecommendationAgent
- **Current Property Logic**: Recommends settlement to property owner (ACV or RCV payout).
- **Motor Insurance Requirements**: Recommends cashless settlement to network garages, reimbursement to policyholders, total loss IDV settlements, and third-party bodily injury/property damage liability settlements.
- **Reusable Logic**: AI decision recommendation recording.
- **Code Changes Required**: Rewrite settlement recommendation prompt to support motor settlement pathways.
- **Estimated Development Effort**: **Medium**

#### 14. PaymentEligibilityAgent
- **Current Property Logic**: Checks claim eligibility against standard property thresholds (limit < $10,000, low complexity).
- **Motor Insurance Requirements**: Minor glass repairs or select cashless garage payments should automatically qualify. Bodily injuries must be excluded from auto-adjudication.
- **Reusable Logic**: Threshold configuration lookups.
- **Code Changes Required**: Implement rules to filter out injury and third-party liability claims from auto-adjudication.
- **Estimated Development Effort**: **Low**

#### 15. PaymentTriggerAgent
- **Current Property Logic**: Triggers disbursement to policyholder bank account or contractor.
- **Motor Insurance Requirements**: Supports cashless disbursements directly to network garages, or reimbursement to the policyholder/third party.
- **Reusable Logic**: Bank transfer webhook triggers, payout status updates.
- **Code Changes Required**: Integrate payment routing fields to select payee (garage vs. policyholder).
- **Estimated Development Effort**: **Low**

---

### 2.4. SIU Persona (12 Agents)

All SIU agents (`BehavioralAnalyticsAgent`, `CaseAssignmentAgent`, `EntityRelationshipAgent`, `EvidenceCorrelationAgent`, `FraudEscalationAgent`, `FraudPatternAgent`, `FraudResolutionAgent`, `FraudRiskScoringAgent`, `LegalEscalationAgent`, `NetworkAnalysisAgent`, `SIUClosureAgent`, `WatchlistUpdateAgent`) can reuse their core LangGraph structures.
- **Assumptions to Modify**:
  - Exclude drone imagery checks and property occupancy details.
  - Add checks for vehicle identification numbers (VIN), RTO ownership records, staged collisions, odometer verification, and telematics location tracking.
- **Code Changes Required**: Rephrase the prompts within the agent directories (`SIUAgents/*Agent/server.py`) to handle motor fraud indicators.
- **Estimated Development Effort**: **Medium** (aggregate for all 12 SIU agents)

---

### 2.5. Vendor Manager Persona (10 Agents)

All 10 Vendor Manager agents (`VendorOnboardingAgent`, `VendorMatchingAgent`, `VendorQualificationAgent`, `VendorCapacityManagementAgent`, `VendorCostBenchmarkAgent`, `DispatchAgent`, `VendorPerformanceAgent`, `SLAComplianceAgent`, `EscalationAgent`, `ETAPredictionAgent`) are architecturally reusable.
- **Assumptions to Modify**:
  - Specializations: Map contractor specialties (Plumbing, Roofing, Electrical) to Auto repair specialties (Body Shop, Mechanic, Paint Shop, Towing Services, Auto Surveyor).
  - Benchmarks: Replace building materials cost benchmarks with OEM/Aftermarket auto parts catalogs and local mechanical labor rates.
- **Code Changes Required**:
  - Update `vendor_matching_mcp/handler.py` to route motor claim categories (e.g., Collision, Glass) to auto garages.
  - Modify `VendorCostBenchmarkAgent` to query vehicle parts databases.
- **Estimated Development Effort**: **Medium** (aggregate for all 10 agents)

---

## 3. Database Changes

To prevent breaking the existing Commercial Property implementation, we must keep the core `claims` and `policy_details` tables generic and introduce LOB-specific extension tables.

```
                  ┌──────────────────────┐
                  │        claims        │
                  └──────────┬───────────┘
                             │ 1:1
              ┌──────────────┴──────────────┐
              ▼                             ▼
   ┌────────────────────┐         ┌────────────────────┐
   │ property_claims    │         │ motor_claims       │
   │ (Property-specific)│         │ (Motor-specific)   │
   └────────────────────┘         └──────────┬─────────┘
                                             │ 1:1
                                             ▼
                                  ┌────────────────────┐
                                  │ vehicle_details    │
                                  └────────────────────┘
```

### 3.1. Existing Tables to Modify

#### 1. `claims`
- **Action**: Generalize and add LOB metadata.
- **Changes**:
  - Add `line_of_business` (TEXT, default 'Property') to distinguish claims.
  - Ensure columns like `location` remain generic (for property address or accident coordinates).
  - Maintain property-specific columns (`police_report_number`) as nullable.

#### 2. `policy_details`
- **Action**: Generalize.
- **Changes**:
  - Add `line_of_business` (TEXT, default 'Property').
  - Ensure `coverage_type` supports values like `Homeowners` and `Motor`.

#### 3. `loss_type_verification_configs`
- **Action**: Add motor loss types.
- **Changes**:
  - Add config rows for: `Collision`, `Auto Theft`, `Glass Damage`, `Third Party Liability`.

---

### 3.2. New Motor-Specific Tables

#### 1. `motor_claims`
- **Purpose**: Holds details specific to motor claims, linked 1:1 to the `claims` table.
- **Columns**:
  - `claim_id` (INTEGER, PK, FK to `claims.id`)
  - `driver_name` (TEXT)
  - `driver_license_number` (TEXT)
  - `driver_license_expiry` (TEXT)
  - `driver_age` (INTEGER)
  - `accident_location` (TEXT)
  - `accident_latitude` (REAL)
  - `accident_longitude` (REAL)
  - `police_fir_number` (TEXT)
  - `police_station` (TEXT)
  - `tow_required` (BOOLEAN)
  - `garage_type` (TEXT - 'Network'/'Non-Network')
  - `cashless_settlement` (BOOLEAN)
  - `injuries_reported` (BOOLEAN)
  - `passenger_count` (INTEGER)
- **Indexes**:
  - Index on `claim_id` (Unique).

#### 2. `vehicle_details`
- **Purpose**: Holds details of the vehicle involved in the claim, linked 1:1 to `motor_claims`.
- **Columns**:
  - `id` (SERIAL, PK)
  - `claim_id` (INTEGER, FK to `motor_claims.claim_id`)
  - `vin` (TEXT)
  - `registration_number` (TEXT)
  - `make` (TEXT)
  - `model` (TEXT)
  - `year` (INTEGER)
  - `engine_number` (TEXT)
  - `chassis_number` (TEXT)
  - `odometer_reading` (INTEGER)
  - `fuel_type` (TEXT)
- **Indexes**:
  - Index on `vin`, `registration_number`.

#### 3. `garage_estimates`
- **Purpose**: Holds structured repair quotes from auto garages.
- **Columns**:
  - `id` (SERIAL, PK)
  - `claim_id` (INTEGER, FK to `motor_claims.claim_id`)
  - `garage_name` (TEXT)
  - `garage_id` (TEXT)
  - `estimate_number` (TEXT)
  - `parts_total` (REAL)
  - `labor_total` (REAL)
  - `tax_amount` (REAL)
  - `grand_total` (REAL)
  - `submitted_at` (TIMESTAMP)

#### 4. `vehicle_inspections`
- **Purpose**: Stores auto surveyor and inspection findings.
- **Columns**:
  - `id` (SERIAL, PK)
  - `claim_id` (INTEGER, FK to `motor_claims.claim_id`)
  - `surveyor_name` (TEXT)
  - `inspection_date` (TEXT)
  - `body_damage_summary` (TEXT)
  - `chassis_aligned` (BOOLEAN)
  - `airbags_deployed` (BOOLEAN)
  - `engine_functional` (BOOLEAN)
  - `roadworthiness_status` (TEXT)
  - `estimated_depreciation` (REAL)

---

## 4. API Changes & External Integrations

To automate validation for Motor Insurance claims, the platform must integrate with several external auto registries and data providers:

| Integration Name | API Endpoint / Provider | Purpose in Motor Claims |
|---|---|---|
| **RTO / DMV Registry** | National Vehicle Registry | Validate registration details, engine/chassis numbers, owner name, and fitness certificate validity. |
| **VIN Decoder** | NHTSA vPIC API | Extract vehicle make, model, year, and trim details automatically from the 17-digit VIN. |
| **License Verification** | National Driver Registry | Verify driver license status, class restrictions, and expiration date. |
| **Telematics / OBD** | OEM Cloud (Tesla, Ford) | Retrieve accident-time speed, deceleration force, airbag triggers, location coordinates, and prior fault codes. |
| **Parts & Labor Database** | Mitchell / Audatex | Benchmark garage repair hours, paint labor costs, and OEM parts prices to prevent inflation. |
| **Surveyor Portal** | Internal Webhook | Dispatch inspection assignments to physical surveyors and capture their standardized repair sheets. |
| **EV Battery Health** | OEM Diagnostic APIs | Assess state-of-health (SOH) and cell temperature of Electric Vehicle batteries to determine total loss risks. |

---

## 5. LLM Prompt Changes

Currently, system prompts hardcode property examples (e.g., "POL-1001", "Homeowners", "water damage from burst pipes"). To support multiple lines of business, prompts should be redesigned to follow one of two strategies:

### Strategy A: LOB-Agnostic Prompt Templates (Recommended)
Prompts are parameterized. The backend wraps system prompts in context blocks that define LOB boundaries before sending them to the LLM.

```
SYSTEM PROMPT TEMPLATE:
You are the Claims Adjuster assistant. Analyze this claim under the {LOB} line of business.
Evaluate the loss using the following constraints:
- Exclusions: {EXCLUSIONS}
- Deductibles: {DEDUCTIBLE}
- Critical Fields: {CRITICAL_FIELDS}
```

### Strategy B: Dynamic LOB-Specific Prompts
The agent server checks the LOB of the claim and queries the corresponding system prompt from Phoenix or local configs.

- **Property Claim**: Loads `prompts/property_adjuster.txt`
- **Motor Claim**: Loads `prompts/motor_adjuster.txt`

---

## 6. Business Rule Changes

| Rule Area | Commercial Property Rule | Motor Insurance Rule |
|---|---|---|
| **Depreciation** | Severity-based flat scales (5% to 30%) or age-based linear depreciation on building systems. | Material-based scales: 50% for rubber/nylon/plastic parts; 30% for glass/fiberglass; metal parts scaled by vehicle age (0% to 50%). Zero-depreciation covers override these scales. |
| **Total Loss** | Total loss declared if building structure is compromised beyond recovery, or business interruption exceeds policy limits. | Constructive Total Loss (CTL) declared if net repair estimate exceeds 75% of the vehicle's Insured Declared Value (IDV). Salvage value is deducted from the payout. |
| **Labor Heuristic** | Labor estimated as a flat 30% of total parts/material costs. | Labor calculated as (labor hours per component * certified mechanic hourly rate) + specialized paint/alignment surcharges. |
| **Exclusions** | Earthquakes, wear and tear, neglect, commercial business interruption periods. | Driving under the influence, driving with an expired license, commercial use of private vehicle, racing, geographical limits violation. |

---

## 7. Validation Changes

1. **Intake Validation**:
   - **Property**: Check for occupancy status, sudden/gradual leak classification.
   - **Motor**: Validate that the driver had a valid license at the time of loss.
2. **Fraud Screening**:
   - **Property**: Weather check for storm/wind claims; drone check for roof manipulation.
   - **Motor**: Verify that telematics-reported impact coordinates align with the reported accident location. Validate VIN against RTO records.
3. **Evidence Validation**:
   - **Property**: Verify contractor invoices against local building code compliance.
   - **Motor**: Match repair invoices with vehicle images showing the damaged parts to identify prior damage fraud.

---

## 8. New Mandatory and Optional Fields

### 8.1. FNOL Intake Fields

| Field Name | LOB | Type | Category | Mandatory / Optional |
|---|---|---|---|---|
| **policy_number** | Shared | TEXT | Policy Details | Mandatory |
| **loss_type** | Shared | TEXT | Loss Details | Mandatory |
| **date_of_loss** | Shared | TEXT | Loss Details | Mandatory |
| **accident_location**| Motor | TEXT | Loss Details | Mandatory |
| **driver_license** | Motor | TEXT | Driver Details | Mandatory |
| **vin** | Motor | TEXT | Vehicle Details | Mandatory |
| **registration_number**| Motor | TEXT | Vehicle Details | Mandatory |
| **make / model / year**| Motor | TEXT | Vehicle Details | Mandatory (or decoded from VIN) |
| **passenger_count** | Motor | INTEGER | Accident Details | Optional |
| **tow_required** | Motor | BOOLEAN | Services | Optional |
| **police_fir_number** | Motor | TEXT | Authorities | Optional |
| **dashcam_footage** | Motor | FILE | Evidence | Optional |
| **telematics_consent** | Motor | BOOLEAN | Consent | Optional |

---

## 9. Configuration Changes

We must implement a configuration engine to load validation rules and mappings dynamically:

```json
{
  "LOB_configs": {
    "motor": {
      "mandatory_fields": ["policy_number", "loss_type", "date_of_loss", "vin", "driver_license", "accident_location"],
      "stp_thresholds": {
        "max_loss_amount": 2500,
        "allowed_categories": ["Glass Only", "Minor Dent"],
        "max_fraud_score": 30
      },
      "vendor_categories": ["Auto Body Shop", "Towing", "Surveyor"],
      "depreciation_waiver": "zero_depreciation_rider"
    }
  }
}
```

---

## 10. Architecture Recommendations

To ensure long-term maintainability, the platform should implement the following design patterns:

### 1. Strategy Pattern for Loss Assessments
Abstract the calculations. Define an interface `LossAssessmentStrategy` with methods `calculate_depreciation`, `calculate_labor`, and `evaluate_total_loss`.
- Implement `PropertyAssessmentStrategy` (flat building calculations).
- Implement `MotorAssessmentStrategy` (material depreciation scales, IDV limits, and CTL checks).

### 2. Rule Engine for Fraud and Triage
Separate business rules from Python code. Use a rule engine (such as `Rule-Engine` in Python) to run JSON-defined rules. This allows business analysts to change fraud indicators and claim scoring rules without modifying agent source files.

### 3. Dynamic Prompt Templates
Use a centralized prompt service (such as Arize Phoenix or a local file manager) that renders LOB-specific segments into base templates before they are passed to the LangGraph execution clients.

---

## 11. Effort & Priority Roadmap

### 11.1. Estimated Agent Development Effort
- **Low Effort (Simple Prompt/Routing Tweaks)**:
  - `ClaimClassificationAgent`, `RoutingAgent`, `PaymentEligibilityAgent`, `PaymentTriggerAgent`, `FeedbackAgent`, `ClaimStatusAgent`, `CommunicationAgent`.
- **Medium Effort (Logic Extension & Conditional Mappings)**:
  - `OrchestratorAgent`, `VoiceTextIntakeAgent`, `DuplicateClaimCheckAgent`, `FNOLOrchestrator`, `DocumentSubmissionAgent`, `ClaimSegmentationAgent`, `TriageAgent`, `EvidenceValidationAgent`, `VerificationAgent`, `ReserveRecommendationAgent`, `FinancialLeakageAgent`, `RepairVsReplacementAgent`, `SettlementRecommendationAgent`, `SIUAgents (all 12)`.
- **High Effort (External API Integrations & Complex Calculations)**:
  - `PolicyCoverageVerificationAgent`, `FraudScreeningAgent`, `ExternalDataAgent`, `DamageAssessmentAgent`, `LossAssessmentAgent`.

---

### 11.2. Priority Implementation Roadmap

```
Phase 1: DB Schema & Intake (Weeks 1-3)
  ├── Generalize core tables (claims, policy_details)
  ├── Create motor_claims and vehicle_details tables
  └── Update VoiceTextIntakeAgent and FNOLOrchestrator
         ▲
Phase 2: Core Verifications & Integrations (Weeks 4-6)
  ├── Integrate RTO, VIN Decoder, and Driver License APIs
  ├── Update PolicyCoverageVerificationAgent
  └── Update ExternalDataAgent and FraudScreeningAgent
         ▲
Phase 3: Damage & Loss Calculations (Weeks 7-9)
  ├── Implement Motor strategy in DamageAssessmentAgent
  ├── Update LossAssessmentAgent and RepairVsReplacementAgent
  └── Integrate Audatex/Mitchell parts benchmarks
         ▲
Phase 4: Orchestration & End-to-End Testing (Weeks 10-12)
  ├── Update OrchestratorAgent (Brain) routing logic
  ├── Refactor SIU and Vendor Manager mappings
  └── Perform end-to-end dry runs (Standard vs. Motor claim pathways)
```
