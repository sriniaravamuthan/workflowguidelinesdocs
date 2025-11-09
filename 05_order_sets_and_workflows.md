# Order Sets and Cross‑Module Workflows

This section describes how clinicians author and use order sets and how the orchestration layer coordinates common patient journeys across multiple HIMS modules.  It also covers task management, decision logic, gates, exception handling and observability.

## Doctor‑Authored Order Sets & Triggers

### Creation & Storage

Clinicians can create, copy, modify and version order sets that include orders, tasks and instructions.  In Cerner, clinicians build **PowerPlans**, collections of orders and clinical content that support frequently used clinical activities; selecting a PowerPlan auto‑populates the selected items【925437338200957†L1935-L1946】.  In Meditech, an order set allows a clinician to select a group of tests or procedures and then review, delete or add individual orders before saving【745034776155115†L919-L950】.  Our system extends this concept by letting authors include decision blocks (conditional logic) and instruction gates.

When creating an order set, the author specifies metadata such as specialty, condition, clinical evidence notes, variants (adult/pediatric) and validity dates.  Authors can save order sets with different sharing scopes:

* **Private:** Only the author sees the order set.
* **Team:** Shared within a group (e.g., Emergency department).
* **Department:** Shared across a specialty (e.g., Surgery service).
* **Hospital‑wide:** Available to all clinicians; may require committee approval.

The system maintains version history.  Edits to an order set create a new version; old versions remain available for audit and for running instances that were started with them.  Authors can view diffs between versions and deprecate old versions with suggested replacements.

### Triggering

Order sets can be applied in several contexts:

* **During an encounter note:** A doctor can apply an order set based on a diagnosis or problem list.  They can preview all items with toggles to include or exclude specific orders and tasks and edit parameters before applying.  This mirrors the ability in Cerner to select PowerPlans from within the Add Orders activity or documentation pages【925437338200957†L1935-L1946】.
* **From the pathway library:** Users can search for order sets by name, condition, tags or variant.  Tags reflect specialties (e.g., *sepsis*, *pre‑operative*).  Favorites can be pinned for quick access.
* **Via NLP authoring:** Clinicians can type free‑text instructions, and the system proposes an order set as described in the NL authoring section.  After confirmation, the order set is applied to the current workflow instance.
* **Automatic triggers:** Decision rules can automatically apply an order set when certain conditions are met (e.g., triage level indicates sepsis).  These triggers use IF‑THEN logic similar to clinical decision support rules【940019348011578†L228-L260】.

### Governance

Hospital‑wide order sets may require committee approval.  Epic’s research order set process involves requests from principal investigators, review by Epic research teams, stakeholder approval and quality control【115630884543560†L46-L105】.  Our governance model includes stages for authoring, reviewing, approving and publishing.  Reviewers check for safety, compliance with local policies and duplication.  Order sets may be deprecated when clinical guidelines change.

## Canonical Cross‑Module Workflows

The orchestration layer coordinates end‑to‑end patient journeys across modules.  Each workflow is defined as a sequence of steps with orders, tasks, instructions, events and timers.

### Outpatient (OPD) Visit

1. **Start:** Triggered when the patient arrives or is auto‑checked‑in through a kiosk or portal.
2. **Collect Copay (Task):** The front desk collects a copay.  This task has an SLA (e.g., 10 minutes) and an escalation policy; if the patient waits too long, the front desk lead is alerted.
3. **Doctor Consultation (Task):** The physician reviews history, performs an exam and records diagnoses.  During this task, the doctor selects relevant order sets.
4. **Apply Order Set (Block):** Orders, tasks and instructions from the selected order set are expanded into the workflow.  For example, an order set for dizziness may include CBC and basic metabolic panel (BMP) orders, an MRI brain if neurological deficits are present, a nurse task to record orthostatic vitals and an instruction on hydration.
5. **Parallel Branches:**
   * **Lab Subflow:** Create lab orders → Collect specimen (task) → Process specimen → Post results → Verify.  The specimen collection subflow may run in parallel to imaging.
   * **Imaging Subflow:** Schedule MRI (task) → Perform MRI → Preliminary report → Final report.  Decision logic may activate this branch only if neuro deficits are recorded.
   * **Counselling/Instructions:** Deliver patient education and collect informed consent where needed.
6. **Wait for Results (Wait Block):** The workflow waits for required results.  Policy settings determine whether it waits for all results or any result (e.g., doctor may proceed after critical labs but before imaging).  The wait block includes a timeout and escalation to ensure that results are not forgotten.
7. **Doctor Review (Task):** The physician reviews results, enters final diagnoses and signs off orders.  Additional orders or tasks may be added if abnormal results are detected.
8. **Pharmacy Dispense (Task):** If prescriptions are present, the pharmacy dispenses medications.
9. **Close Visit (Task):** Billing reviews charges, generates an invoice and issues a visit summary.  Disposition is recorded as follow‑up, referral or admit.
10. **Compensations:** If the visit is abandoned (no‑show or left against medical advice), copays are refunded, orders are cancelled and bookings are released.  The system handles these compensations automatically.

### Inpatient Admission

1. **Pre‑admission:** A tentative encounter is created; initial patient information and estimated charges are captured.
2. **Bed Allocation:** Decision logic selects an appropriate ward or isolation room based on diagnosis, infection status and bed availability.  A task is created to reserve the bed.
3. **Admission Orders:** The admitting provider applies an order set containing baseline labs, medications, venous thromboembolism (VTE) prophylaxis, diet and nursing tasks.  Variants may exist for adult vs. pediatric or high‑risk vs. low‑risk admissions.
4. **Parallel Care:** Nursing flowsheets, medication administration and specialist consults run concurrently.  Each has its own tasks and SLAs.
5. **Procedure/Operating Room (Optional Subflow):** If the patient requires surgery, pre‑operative checklists, case carting, WHO sign‑in and anesthesia documentation are executed as subflows.
6. **Discharge Ordered (Gate):** The physician indicates intent to discharge.  This gate blocks further steps until clearances are obtained.
7. **Clearances:** Pharmacy ensures medications and devices are returned; billing finalizes charges; case management arranges follow‑up.  Each clearance may be a task with its own SLA and escalation.
8. **Discharge Execution:** The ADT module changes the patient’s status to discharged.  A discharge summary is generated and provided to the patient.
9. **EVS Turnover:** Environmental services receive a task to clean and prepare the bed for the next patient.  Once the bed is ready, a *Bed Ready* event ends the workflow.

### Emergency/Trauma (Fast‑Track)

1. **Quick Registration:** Minimal demographics are captured to create a temporary record.
2. **Triage (Task):** A nurse assigns an Emergency Severity Index (ESI) level.  Decision logic routes the patient to appropriate pathways (e.g., resuscitation, urgent care, minor injuries).
3. **Immediate Orders:** For high‑acuity patients, the system automatically applies a trauma order set (e.g., FAST ultrasound, X‑ray, type & screen, blood transfusion protocols).  Tasks such as cervical spine immobilization and consent are created.
4. **Wait for Critical Results:** Tight timers monitor time to key results (e.g., blood gas) and raise alerts if delays occur.
5. **Branching:** Disposition may include discharge, admission to ICU/ward or transfer to operating theatre.  Handover tasks convey context to receiving units.

### Diagnostic Workflow (Lab & Imaging)

* **Lab:** Orders are created; specimens are labelled and tracked; processing occurs; results are posted and verified; order statuses progress from Logged to Complete to Resulted【745034776155115†L979-L1016】.  Tasks include specimen collection and result verification.
* **Imaging:** Orders are protocolled by a radiologist; scheduling tasks assign a slot; imaging is performed; preliminary and final reports are generated.  Decision logic can block contrast imaging until renal function is acceptable or a nephrologist clears the case.

### Discharge Planning

* **Discharge Intent:** The provider indicates an intent to discharge early in the admission so planning can begin.
* **Checklist Gate:** A gate requires completion of tasks such as medication reconciliation, patient education, device returns, consult closures and follow‑up appointments.
* **Billing Finalization:** Charges are reviewed; pending balances are resolved or financial plans are agreed.
* **Discharge Execution:** The ADT module updates the patient’s status; a discharge packet is provided to the patient.  EVS is notified to turn over the bed.

## Task Management: Forms, Inbox and SLA

### Task Forms

Each task is associated with a **form** that captures required information.  Forms include fields with validations, drop‑downs, and helper text.  They may support attachments (photos of wounds, consent forms) and automatically display context such as patient name, MRN, allergies and location.  The front‑end design should aim for minimal keystrokes and clear guidance for novices.

### Inbox & Queues

Tasks appear in **inboxes** or **queues** based on role, team or department.  Sunrise’s Workflow Management tool allows clinicians to assign workflows, view status of activities and complete them【736777591292277†L16-L22】.  Similarly, our engine provides:

* **My Tasks:** Tasks assigned to the current user.
* **Team Queue:** Unassigned tasks that any team member can pick up.
* **Department Queue:** Broader queue for a department (e.g., all pending lab draws).
* **Aging Buckets:** Filters showing tasks approaching or breaching their SLA.
* **Bulk Actions:** Where safe, users can acknowledge or reassign multiple tasks.

### SLA, Reminders & Escalations

Every task has an SLA.  Reminders are sent before deadlines; escalations occur if tasks remain incomplete.  Escalation notifications may go to supervisors (e.g., charge nurse) or cross‑functional leads.  Escalation logs capture reasons and responses.  The system can integrate with on‑call scheduling to ensure notifications reach the right person.

## Decisions & Rules

Decision logic determines which branch to follow, which variant of an order set to apply and whether certain actions are allowed.  **Rules** derive from clinical guidelines, operational policies and payer requirements.  For example:

* **Clinical rules:** If a patient is on anticoagulants, hold them before surgery.  Discern Expert in Cerner builds rules that evoke when events occur (placing an order, charting a condition), evaluate patient data and then perform actions such as suggesting a PowerPlan【925437338200957†L1967-L1984】.
* **Operational rules:** If a bed is not available, hold the admission and notify the bed manager.  If copay is not collected within a grace period, cancel the appointment.
* **Payer rules:** If prior authorization is required for imaging, create a task to obtain authorization before scheduling.

Rules can detect conflicts (duplicate orders, contraindications) and enforce prerequisites.  Clinical decision support literature notes that rules in knowledge‑based systems evaluate data through IF‑THEN statements and trigger actions or alerts【940019348011578†L228-L260】.  Our rule library supports such logic while allowing authors to build and test rules in a controlled environment.

## Checklists, Gates & Compliance

**Gates** represent formal checkpoints in a workflow.  They bundle multiple requirements that must be satisfied before proceeding.  For example, a **Pre‑operative Gate** may require completed consent, site marking, pre‑op labs, anesthesia assessment and WHO surgical safety checklist.  Each requirement is either a task or an attested item (with timestamp and responsible role).

Checklists map to quality standards (e.g., WHO surgical safety, NABH accreditation).  Completion of checklists can be audited and reported.  Gates may allow overrides with justification, but overrides are always recorded and flagged for review.

## Exception Handling & Compensations

Workflows must anticipate and handle failures.  **Failure classes** include:

* **Transient:** Temporary issues (network outage, service unavailable) that can be retried.
* **Terminal:** Irrecoverable errors (patient left the facility, lab instrument failure) that require cancellation.
* **Policy:** Violations of clinical or operational policy (allergy conflict, missing consent) that require escalation.
* **Manual:** Cases requiring human intervention (e.g., ambiguous data, conflicting orders).

For each step, authors define **compensation actions** to undo or offset side effects: cancel appointments, reverse charges, release beds, void medication labels, rescind instructions and notify recipients.  Safe **resume points** exist at every wait or gate so that workflows can pause and later resume.

**Abandonment** is explicitly declared when a patient does not show up or leaves against medical advice.  The system records the reason (no‑show, financial, personal) and performs compensations such as releasing resources and notifying staff.

## Observability & Audit

The orchestration layer must be observable.  It captures a **timeline** for each instance, listing who performed each action, when it occurred and what the outcome was.  Key metrics include:

* **Cycle time:** Total duration from start to completion.
* **SLA breach percentage:** Proportion of tasks that exceeded their SLA.
* **Wait times:** Time spent waiting for events or results.
* **Cancellation and compensation rates:** Frequency of cancelled orders and triggered compensations.
* **Queue depth:** Number of tasks waiting in each queue.

Users can view dashboards showing live backlogs and bottlenecks.  Histories are immutable for sensitive actions such as overrides and compensations.  Reports identify trending issues (e.g., recurring SLA breaches in phlebotomy) and support continuous improvement.

By providing robust order set authoring, clearly defined workflows for common clinical scenarios, rigorous task management, rule‑based decisions, formal gates, comprehensive exception handling and rich observability, the orchestration layer aligns with and extends the capabilities found in leading EMRs while maintaining a user‑friendly, business‑configurable approach.
