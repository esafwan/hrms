# Huf + Frappe HRMS Implementation Reference for AI Agents

## 1) Purpose and Scope
This document is a **build-time and run-time reference** for AI agents working in the `huf` codebase to implement HRMS-specific tools, automations, and features for Frappe HRMS environments.

It is intentionally prescriptive: it defines naming, APIs, method contracts, data-access boundaries, workflow triggers, governance controls, and rollout order so an agent can act confidently with minimal ambiguity.

**Primary objective:**
- Treat **Frappe HRMS as system of record**.
- Use **Huf as orchestration + intelligence layer**.
- Keep all AI actions permission-aware, auditable, and approval-gated for sensitive actions.

---

## 2) Source References (Authoritative Inputs)
Use these as first-priority references while implementing in `huf`:

1. Huf documentation home (concepts, guides, architecture): https://docs.huf.ai/
2. Huf tools concept page (tool types + execution flow): https://docs.huf.ai/docs/concepts/tools/
3. Huf tool publishing from Frappe apps (`@agent_tool` pattern): https://docs.huf.ai/docs/tools/publishing/
4. Huf development patterns (including tool wrapper pattern): https://docs.huf.ai/docs/development
5. Huf GitHub repository: https://github.com/tridz-dev/huf
6. Frappe HRMS repository (DocTypes, workflows, HR data model): https://github.com/frappe/hrms
7. Frappe framework docs (permissions, whitelisted methods, DocType events, scheduler): https://frappeframework.com/docs

> Implementation rule: if this document conflicts with current `huf` runtime APIs, follow current `huf` code and update this document as part of the same change set.

---

## 3) Architecture Positioning

### 3.1 System-of-Record Boundary
- **HRMS owns canonical data**: Employee, Leave, Attendance, Salary Slip, Expense Claim, Appraisal, Recruitment, Onboarding/Separation records.
- **Huf owns intelligence + orchestration**: natural language understanding, tool routing, summaries, draft generation, controlled workflow assistance.

### 3.2 AI Capability Layers
1. **AI Essentials (Phase 1)**: mostly read-only, explanatory, and draft-first.
2. **Workflow Automation (Phase 2)**: trigger/schedule-driven assistants with guarded write actions.
3. **Advanced Intelligence (Phase 3)**: recruitment/appraisal analytics, attrition signals, service desk, strategic summaries.
4. **Governance (All phases)**: RBAC, logs, approval gates, feedback loops, PII controls.

---

## 4) Huf Tooling Pattern to Follow

Huf supports built-in tools and app-published tools. For HRMS integration work, prefer **published custom tools** with narrow, explicit responsibilities.

### 4.1 Tool Design Rules
- One business capability per tool.
- Prefer deterministic output schema.
- Never hide permission decisions in prompt text only; enforce in Python.
- Use small composable tools rather than one broad “do everything” tool.
- Return actionable diagnostics for failures (`reason_code`, `message`, `next_step`).

### 4.2 Suggested Tool Naming Convention
Use `hrms_<domain>_<action>`:
- `hrms_leave_get_balance`
- `hrms_payroll_explain_payslip`
- `hrms_attendance_get_exceptions`
- `hrms_onboarding_create_checklist`

### 4.3 Suggested Python Function Path Convention
Within an integration app/module:
- `huf_hrms.tools.leave.get_balance`
- `huf_hrms.tools.payroll.explain_payslip`
- `huf_hrms.tools.attendance.get_daily_exceptions`

### 4.4 Suggested Tool Metadata Contract
Each tool definition should include:
- **name**
- **description** (high-signal, decision-useful)
- **input_schema** (typed)
- **output_schema** (typed)
- **required_roles**
- **sensitivity_level** (`low`, `moderate`, `high`)
- **approval_required** (boolean)
- **idempotent** (boolean)

---

## 5) Canonical HRMS Domains and DocTypes (Mapping Layer)

Use this mapping when designing tool coverage.

- Employee Core: `Employee`, `Department`, `Designation`, `Company`
- Leave: `Leave Type`, `Leave Allocation`, `Leave Application`, `Leave Ledger Entry`
- Attendance: `Employee Checkin`, `Attendance`, `Shift Type`, `Shift Assignment`
- Payroll: `Salary Structure`, `Salary Slip`, `Payroll Entry`, `Additional Salary`, `Loan`, `Employee Advance`
- Expense: `Expense Claim`, `Employee Advance`
- Recruitment: `Job Opening`, `Job Applicant`, `Interview`
- Performance: `Appraisal`, `Goal`, (org-specific KRAs as custom DocTypes)
- Onboarding/Offboarding: `Employee Onboarding`, `Employee Separation`, task templates/checklists
- Compliance docs (org/custom): passport, visa, national ID, labor card, contract expiry fields

> Note: DocType availability can differ by HRMS version and customization. Tool code must fail safely when optional fields/DocTypes are absent.

---

## 6) Phase 1 Tool and Agent Pack (Minimum Viable, High-Impact)

## 6.1 Employee HR Assistant (Read + Explain)
Tools:
- `hrms_leave_get_balance(employee, as_of_date?)`
- `hrms_leave_get_applications(employee, status?, from_date?, to_date?)`
- `hrms_payroll_get_latest_payslip(employee)`
- `hrms_payroll_explain_payslip(employee, salary_slip)`
- `hrms_expense_get_claim_status(employee, claim_id?)`
- `hrms_policy_answer(query, policy_scope?)`

Outputs should be plain-language and include source record IDs.

## 6.2 HR Policy Knowledge Assistant
- Index approved policy documents only.
- Always return citations to document title + section/page chunk.
- If policy answer confidence is low, return “needs HR validation”.

## 6.3 HR Admin Assistant
Tools:
- `hrms_employee_find_missing_documents(document_type, department?, branch?)`
- `hrms_employee_find_expiring_documents(document_type, within_days=60)`
- `hrms_leave_get_pending_approvals(approver_user)`
- `hrms_payroll_find_missing_salary_structures(company?, payroll_period?)`

## 6.4 Management Assistant (Aggregate Only by Default)
Tools:
- `hrms_headcount_by_department(company, as_of_date?)`
- `hrms_absenteeism_summary(period, dimension='department')`
- `hrms_join_exit_summary(period, dimension='department')`
- `hrms_payroll_trend(period_granularity='month', months=12)`

Default to aggregated outputs to minimize privacy leakage.

---

## 7) Phase 2 Workflow Agents (Trigger/Schedule Driven)

## 7.1 New Employee Onboarding Agent
Trigger:
- Employee created OR onboarding started.
Actions:
- completeness check
- checklist generation
- stakeholder notifications

## 7.2 Leave Approval Intelligence Agent
Trigger:
- Leave Application submitted.
Actions:
- balance/policy validation
- team overlap check
- impact summary + draft decision note

## 7.3 Expense Claim Review Agent
Trigger:
- Expense Claim submitted.
Actions:
- required receipt checks
- policy mismatch flags
- draft clarification note

## 7.4 Attendance Exception Agent
Trigger:
- daily schedule after check-ins sync.
Actions:
- missing check-ins
- repeated late/early patterns
- manager/employee reminders

## 7.5 Payroll Readiness Agent
Trigger:
- scheduled pre-payroll window.
Actions:
- missing attendance
- unapproved leaves
- missing salary structure
- unresolved loans/advances/additional salary items
- readiness checklist

## 7.6 Offboarding Agent
Trigger:
- separation/resignation initiated.
Actions:
- handover checklist
- asset/loan/advance/leave summary
- final settlement readiness note

## 7.7 Compliance Expiry Agent
Trigger:
- daily/weekly scan.
Actions:
- upcoming expiry notices
- overdue escalations

---

## 8) Phase 3 Advanced Intelligence Pack

- Recruitment summarization and match support (assistive only)
- Appraisal summarization and feedback drafting (no autonomous rating)
- HR ticket triage assistant
- Workforce trend and anomaly summaries
- Attrition signal summaries (non-deterministic, no individual hard prediction claims)

---

## 9) Security, Permissions, and Governance (Non-Negotiable)

## 9.1 Permission Model
- Use Frappe permission checks in tool methods.
- Enforce row-level access rules:
  - employee => self data
  - manager => scoped team data
  - HR/payroll/admin => role-defined broader access

## 9.2 Sensitive Actions Requiring Human Approval
Always approval-gated:
- payroll submit/finalize
- salary structure changes
- terminations/separation finalization
- final settlement approval
- final appraisal rating commit

## 9.3 Audit Event Schema (Mandatory)
Log each run with:
- `run_id`, `agent`, `user`, `timestamp`
- `input_summary`
- `tools_called[]`
- `records_accessed[]`
- `response_summary`
- `proposed_actions[]`
- `executed_actions[]`
- `token_usage`, `model`, `cost_estimate`
- `approval_state`

## 9.4 Privacy Controls
- Mask national ID/passport numbers in conversational outputs.
- Show salary values only to authorized roles.
- Avoid exposing personal records in group-channel contexts.

---

## 10) API and Method Templates (Reference Contracts)

```python
# Example contract for discovered tool
from typing import TypedDict, Optional

class LeaveBalanceInput(TypedDict):
    employee: str
    as_of_date: Optional[str]

class LeaveBalanceOutput(TypedDict):
    employee: str
    as_of_date: str
    balances: list[dict]
    source_records: list[str]

# pseudo-decorator name; adapt to actual huf discovery decorator in use
@agent_tool(name="hrms_leave_get_balance", description="Get leave balances for an employee")
def get_leave_balance(input: LeaveBalanceInput) -> LeaveBalanceOutput:
    # 1) enforce user permission context
    # 2) query Leave Allocation / Leave Ledger
    # 3) return structured response
    ...
```

Implementation checklist per tool:
1. Validate inputs (type + semantic constraints).
2. Resolve acting user context.
3. Enforce permission checks.
4. Query DocTypes with filters/limits.
5. Return deterministic schema.
6. Emit audit event.
7. If write action: require approval token/state.

---

## 11) Prompting and Response Standards for HR Agents

- Use concise operational language.
- Distinguish facts vs inference.
- Always mention the period used (“April 2026”, “last 30 days”).
- For policy answers, include citation snippet reference.
- For uncertainty, ask for one clarifying datum rather than guessing.
- Never provide legal/tax advice unless organization-approved policy source explicitly covers it.

Recommended response template:
1. Direct answer
2. Key details (bullets)
3. Source records/policies used
4. Suggested next action

---

## 12) Error Handling and Guardrails

Standard `reason_code` values:
- `PERMISSION_DENIED`
- `MISSING_RECORD`
- `MISSING_CONFIGURATION`
- `POLICY_UNAVAILABLE`
- `APPROVAL_REQUIRED`
- `VALIDATION_ERROR`
- `TRANSIENT_FAILURE`

For each failure return:
- what failed
- why it failed
- exact remediation step (user or HR admin)

---

## 13) Rollout Sequence and Acceptance Criteria

## 13.1 Recommended Rollout
1. Read-only Q&A and summaries.
2. Draft recommendations.
3. Approval-gated actions.
4. Bounded automation.
5. Advanced predictive/analytic support.

## 13.2 KPI Set
- AI-resolved HR queries / total HR queries
- payroll exceptions found pre-run
- approval turnaround reduction
- missing document reduction
- onboarding SLA adherence
- AI answer satisfaction score

---

## 14) Build Backlog (Concrete Tickets for Huf Agent)

1. Create HRMS tool package scaffold:
   - `huf_hrms/tools/{leave,payroll,attendance,expense,onboarding,offboarding,policy}.py`
2. Implement 8 Phase-1 read/explain tools with strict schemas.
3. Add HR policy retrieval utility with citation chunking.
4. Add approval-gate middleware for sensitive tools.
5. Add audit logger sink for each agent run and tool call.
6. Add scheduled agent definitions (attendance exceptions, payroll readiness, compliance scan).
7. Add test fixtures for permission boundary scenarios.
8. Add dashboard cards for AI run count, failure codes, pending approvals, and confidence flags.

---

## 15) Definition of Done for Any HRMS Tool
A tool is complete only when:
- role/permission checks verified,
- response schema documented and tested,
- audit logs emitted,
- edge cases covered (missing config, missing records),
- prompt/tool description is clear enough for autonomous tool selection,
- sensitive action path has explicit approval requirement.

---

## 16) Final Positioning Statement (Reusable)
Frappe HRMS remains the enterprise HR system of record, while Huf provides a role-aware, workflow-aware, auditable AI layer that helps employees, HR teams, managers, and leadership understand data, prepare actions, and automate defined HR workflows safely.
