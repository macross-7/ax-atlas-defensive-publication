# Defensive Publication: AX Atlas — Integrated SaaS System for AI-Adaptive Workflow Interview, Automation Catalog Generation, and Customer-Specific Multi-Tenant Dashboard Deployment

> **Type of disclosure:** This document is a **Defensive Publication** intended to establish prior art and prevent third parties from obtaining patents on the disclosed subject matter. The author may, at the author's sole discretion, file a corresponding patent application with the Korean Intellectual Property Office (KIPO) or any other patent office, but is not obligated to do so. This publication is irrevocable upon posting and stands independently as prior art regardless of any future patent filing.
>
> **Date of first public disclosure:** 2026-05-10 (UTC). The Git commit timestamp on the corresponding GitHub repository serves as the authoritative prior-art date.
>
> **Authors / Inventors:** 김현구 (Hyungu Kim).
>
> **Corresponding patent application (if any):** **Not yet filed (출원 전).** As of the date above, no patent application corresponding to this disclosure has been filed with any patent office.

---

## 1. Abstract

This disclosure describes an **integrated SaaS system** that combines (i) AI-adaptive natural-language interviews of enterprise employees, (ii) automated conversion of interview results into standardized **Implementation Blueprints** (a six-field automation specification), (iii) **multi-tenant customer dashboards** with isolated workspaces, (iv) **dynamic deployment** of automation deliverables as menu/feature items into each customer's dashboard, (v) **operational KPI measurement** of deployed automations across multiple metrics, and (vi) a **closed-loop promotion mechanism** that re-interviews task owners and automatically upgrades the automation tier based on accumulated operational data.

The combination of these six elements into a single platform — including in particular the closed-loop promotion of human-only tasks into AI-assisted or fully-automated tasks based on measured KPIs — has not, to the inventor's knowledge, been disclosed in any single prior art reference.

## 2. Technical Field

Enterprise workflow automation analysis and dynamic automation-feature deployment systems; AI-driven business process discovery; multi-tenant SaaS platforms with embedded automation runtime.

## 3. Background and Problem

Existing approaches each have structural limits:

- **External consulting**: 3–6 months of interviews produce static reports; results rarely become executable assets and do not persist as a living catalog inside the company.
- **Process / Task Mining (e.g., Celonis, UiPath Task Mining)**: limited to work that already produces system logs or desktop telemetry; cannot capture human judgment, phone calls, paper-based work, or stated pain points.
- **AI interview tools (e.g., HireVue)**: confined to recruiting; no enterprise-wide work catalog application; follow-up questions are simple branching rather than depth-driven.
- **AI dashboard generators**: limited to data visualization; do not deploy *built automation deliverables* as runnable features into per-tenant workspaces.
- **None of the above** measure the operational performance of deployed automations or feed those measurements back into a systematic re-classification and promotion loop.

## 4. Solution Architecture (Four Core Modules + Two Operational Layers)

### Core modules

1. **AI Adaptive Interview Module**
   - Conducts natural-language dialogue with each employee.
   - Scores answer depth across four dimensions: `specificity`, `process`, `judgment`, `pain_points`, summed on a 0–100 scale (**Depth Score**).
   - Verifies extraction coverage across **16 analysis axes**; treats the interview as completable when coverage ≥ 90%.
   - When Depth Score is below a configurable threshold, dynamically generates follow-up **Probe** questions until depth is achieved.

2. **Blueprint Generation Module**
   - Converts each interview result into a **six-field Implementation Blueprint**: `system_architecture`, `tools`, `inputs`, `automated_steps`, `outputs`, `ai_prompt_md`.

3. **Customer Multi-Tenant Dashboard Module**
   - Each customer enterprise receives an isolated tenant identifier and **per-customer KMS BYOK encryption keys**.
   - Users are assigned to a **five-level role hierarchy (L1–L5)**; menu visibility and feature access are differentiated per level.

4. **Dynamic Menu Deployment Module**
   - Upon receipt of an inspection-PASS signal for a given automation deliverable, the system **immediately adds** that deliverable to the target customer's dashboard as a runnable menu/feature, transitioning the deliverable to a "Live" state.
   - The full lifecycle traverses seven workflow states: (1) AI initial estimate, (2) work request, (3) firm estimate, (4) acceptance, (5) development, (6) inspection PASS, (7) Live. The firm estimate is constrained to be ≤ the AI initial estimate.

### Operational layers (the closed loop)

5. **Operational KPI Measurement Layer**
   - Once a deliverable is Live, the platform records **six operational metrics** as time series, aggregated per Tier:
     `executions_count`, `success_rate`, `avg_processing_time`, `cost_per_execution`, `human_intervention_rate`, `review_ng_rate`.
   - Each Tier is assigned an operational state label — `HEALTHY`, `WARNING`, or `UNHEALTHY` — by comparing the metrics to configurable thresholds.

6. **Closed-Loop Promotion Layer** _(novel)_
   - When any of the following triggers fires — (a) a configurable time interval (e.g., 90 days) has elapsed since the most recent classification, (b) the operational state has transitioned to `UNHEALTHY`, or (c) the customer's scheduled review period arrives — the system **automatically re-runs the adaptive interview** for the relevant task.
   - The classification score is recomputed; if it now meets the threshold of a higher automation tier, the system **promotes** the task one or more levels (e.g., `human_only → ai_assist → full_auto`). Demotion follows the same mechanism in reverse when KPIs deteriorate.
   - The capacity recovered from a promoted task is **automatically reallocated** to other un-automated tasks ranked by:

     ```
     priority_score(task_i) = business_impact(task_i)
                             × automation_feasibility(task_i)
                             × (1 − current_automation_coverage(task_i))
     ```

This closed loop converts the platform from a one-shot classifier into a **self-improving automation portfolio**.

## 5. Auxiliary Mechanisms

- **Tier auto-classification**: each automation case is scored across seven variables (input data variety, external system integrations, LLM call patterns, reliability requirements, SLA requirements, training data preparation, business impact), and the sum is mapped to **five tiers (T1–T5)** ranging from self-service to project-grade engagements.
- **Pricing auto-derivation**: the AI-issued initial estimate equals `standard_person_days × 1.25 (safety buffer) × unit_rate`; the firm estimate is constrained to be ≤ the initial estimate.
- **Refund auto-derivation on inspection-NG**: the system counts inspection-NG events; once the configured free-revision quota is exceeded, refund ratios (e.g., 90% / 70% / 50% / 30% / 20% / 0%) are applied automatically based on the workflow stage.
- **MSP auto-enrollment on go-live**: when a deliverable transitions to Live, the system automatically calculates a recurring monthly managed-service fee as `development_cost × tier_specific_rate` and begins billing, converting one-shot project revenue into a recurring revenue stream.

## 6. Embodiment Data (validated)

In a Cycle-0 deployment to a Korean enterprise of approximately 1,000 employees, the system achieved (representative figures):

- Mean Depth Score: ≈ 74 / 100
- 16-axis extraction coverage: ≈ 92%
- Quick-Win identifications per department: ≈ 2.4
- `verified_by_owner` rate: ≈ 88%
- Interview participation rate: ≈ 76%
- Variable cost per interviewed employee: ≈ USD 0.65 (using a mainstream LLM stack on a serverless edge runtime)

These figures evidence the input → processing → output correlation required by KR/JP patent practice for AI-related inventions.

## 7. Variant Embodiments (Equivalents)

The disclosure expressly contemplates the following equivalents and intends to establish prior art covering each of them:

- **LLM substitution**: Anthropic Claude, OpenAI GPT, Google Gemini, Mistral, in-house fine-tuned models, or multi-LLM ensembles (majority vote / routing).
- **Depth Score dimensionality**: 3–10 dimensions, with weights varying by industry, department, or country.
- **Analysis axes**: 4–64 axes; weighting differentiated per industry, department, or country.
- **Automation classification levels**: 2–10 levels (e.g., 3-level full_auto/ai_assist/human_only; 5-level full_auto/ai_assist_high/ai_assist_low/human_lead/human_only; 7-level fine-grained variants).
- **Classification mechanism**: rule-based five-question boolean classification, machine-learning classifier (random forest, XGBoost, neural net), or LLM self-judgment.
- **Blueprint fields**: 3–15 fields, including optional fields for test scenarios, monitoring metrics, security requirements, or compliance requirements.
- **Deployment channels**: dashboard menu, REST API endpoint, RPA workflow node (UiPath / Power Automate), chatbot slash command (Slack / Teams), email-triggered automation, scheduled cron execution, A/B canary deployment with auto-rollback on KPI degradation.
- **Role models**: 3-tier / 5-tier / 7-tier hierarchies, RBAC matrices, or attribute-based access control (ABAC).
- **Pricing models**: flat fee, development-cost percentage (with tier-differentiated rates), usage-based (`base_fee + unit_price × executions_count`), performance-linked (`base_fee × min(success_rate / target, 1.0)`), or hybrid weighted combinations.
- **Tier mapping thresholds**: differentiated by industry, country, or company size (e.g., more conservative thresholds for public-sector customers).

Any implementation that maintains the closed loop **interview → blueprint → deployment → KPI measurement → re-classification → promotion / capacity reallocation** is considered an equivalent of the disclosed invention for purposes of this defensive publication.

## 8. What is *not* disclosed

The following are intentionally **not** disclosed in this publication and remain the author's trade secrets and operational know-how: production credentials, infrastructure topology, internal customer data, and specific commercial pricing tables.

## 9. Notice

This publication is irrevocable. By posting it, the author dedicates the disclosed combinations to the public record for the purpose of defeating later patent applications by third parties on substantially the same subject matter, in any jurisdiction.

If a patent application by the author corresponding to this disclosure is filed and later abandoned without registration, the contents of this publication remain in the public domain.

---

_End of defensive publication._
