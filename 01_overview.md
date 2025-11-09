# Purpose, Scope and Vision

## Purpose & Vision

This document describes a functional workflow‑orchestration layer that sits on top of a hospital information management system (HIMS).  The goal is to provide clinicians and operations teams with a **visual, business‑configurable engine** for ordering, tasking and coordinating care across appointments, registration, billing, doctor consultation, laboratory and imaging services, pharmacy, admission/discharge/transfer (ADT), environmental services and other modules.  The engine should enable care teams to assemble **order sets** and **clinical pathways** without writing code and to ensure that every step is deterministic, auditable and safe.

In mature electronic medical record (EMR) platforms such as Epic and Cerner, the creation of bundled orders and workflows is part of daily practice.  For example, Cerner’s **PowerPlan** feature allows clinicians to build collections of orders and clinical content that support common interventions; they can be selected from within order entry or documentation pages and automatically populate the appropriate details【925437338200957†L1935-L1946】.  Similarly, Meditech’s order entry module describes an **Order Set** as a group of tests or procedures commonly ordered together; when a clinician selects an order set, all orders in the preset automatically default into the entry screen and the default responses can be reviewed and edited【745034776155115†L919-L950】.  Our orchestration layer adopts this concept, generalizing it to include not just orders but also tasks, instructions, decisions and events.

## Scope

### In scope

The orchestration layer covers functional requirements for:

* **Workflow modeling:** definitions of workflows, states, transitions, events and timers.  The system stores versioned blueprints for processes such as outpatient visits, admissions or imaging procedures.
* **Visual designer:** a drag‑and‑drop canvas with a palette of domain‑specific blocks.  Clinicians can assemble flows without coding and inspect each block’s parameters.  Validation ensures there are no dead ends or unbounded waits.
* **Natural‑language authoring:** a mechanism to translate free‑text instructions into structured order sets and workflows.  The system parses user input, identifies orders, tasks and instructions, and prompts the author for any ambiguous elements before converting the result into a reusable template.
* **Order set semantics:** support for bundled orders, tasks and instructions, including conditional activation and variants.  This reflects how commercial EMRs allow users to define multi‑phase plans of care, such as Cerner’s PowerPlans【925437338200957†L1935-L1945】.
* **Cross‑module orchestration:** deterministic step‑to‑step flows across appointments, billing, consultation, ancillary services, pharmacy, ADT and discharge.  Waits and timers govern parallel branches and ensure that each step completes or escalates.
* **Human task management:** inboxes and queues for tasks that require human action.  Sunrise’s Workflow Management tool, for example, lets clinicians assign workflows, view activity status and complete tasks【736777591292277†L16-L22】; our engine should offer similar capabilities with configurable service‑level agreements (SLAs) and escalations.
* **Exception handling and compensations:** defined compensation actions to undo or mitigate side effects when a step fails or a patient journey is abandoned.
* **Observability:** audit trails, metrics and dashboards showing cycle times, SLA breaches and bottlenecks.
* **Versioning and governance:** change control for workflow definitions and order sets; drafts may be reviewed and approved before they are published.

### Out of scope

The functional specification deliberately excludes detailed role‑based access control (RBAC) policy grammar, implementation details such as database schemas or API specifications, and low‑level code.  It focuses on what the system must do rather than how it will be built.

## Design Tenets

1. **Clinical native modelling:** First‑class objects include Orders, Tasks, Instructions, Order Sets, Encounters and Events.  This mirrors the familiar constructs in commercial EMRs and avoids exposing general‑purpose BPMN or technical terminology to clinicians.
2. **Business‑first configuration:** The designer uses **natural verbs** and domain blocks (e.g., “Collect Specimen”, “Assign Nurse”) rather than programming concepts.  Users can compose flows from these blocks much like they select prebuilt PowerPlans or Order Sets in Cerner【925437338200957†L1935-L1945】 or Meditech【745034776155115†L919-L950】.
3. **Deterministic runtime:** Every transition is explicit, every wait has a timeout and every action is idempotent.  This ensures predictable behavior and allows safe compensation for failures.
4. **Composable subflows:** Reusable subflows (e.g., a “Collect Specimen” sequence) can be nested within larger flows, enabling reuse and simplifying maintenance.
5. **Traceable execution:** The system maintains a full breadcrumb of who did what and when.  Meditech’s order management screens allow users to see the status of each order (Verified, Transmitted, In Process, Cancelled, etc.)【745034776155115†L979-L1016】; similarly, the orchestration layer will provide step‑level metrics and audit logs.
6. **Safety by default:** Risky actions require confirmations.  Compensation routines are specified upfront to undo irreversible effects.  Clinical decision support rules (knowledge‑based IF‑THEN rules) evaluate patient data to trigger alerts or recommendations【940019348011578†L228-L260】.
7. **Clinician empowerment:** Doctors and nurses can author, save and share order sets and workflows.  Epic order set processes involve clinicians submitting requests, reviewing builds and approving final screens【115630884543560†L46-L105】; our engine seeks to make authoring more self‑service but still respects governance.

## Definitions & Actors

| Term | Explanation |
|---|---|
| **Workflow Definition** | A versioned blueprint describing a process (e.g., *OPD Visit v4*). Each definition has states, transitions, timers and compensation logic. |
| **Workflow Instance** | A running case of a definition bound to a particular patient, visit or encounter. |
| **Order** | A clinically actionable request such as a laboratory test, imaging study, medication, diet or procedure.  In Meditech, orders may be single, series or part of order sets【745034776155115†L164-L172】. |
| **Task** | A unit of work requiring human action, such as collecting payment, drawing blood or performing counselling.  Tasks have assignee roles, input forms, output fields, SLAs and escalation policies.  Sunrise’s Workflow Management tool lets clinicians view and complete workflow activities【736777591292277†L16-L22】. |
| **Instruction** | Non‑order guidance to the care team or patient.  Instructions may be blocking (e.g., *“Do not discharge until MRI reviewed”*) and can include patient education or isolation precautions. |
| **Order Set** | A bundle containing orders, tasks and instructions.  EMRs like Cerner call these **PowerPlans**; they expedite common interventions and can be selected during ordering【925437338200957†L1935-L1946】.  Meditech’s order sets automatically populate a series of orders and allow clinicians to edit defaults【745034776155115†L919-L950】. |
| **Event** | A signal that advances a workflow, such as *Patient Arrived*, *Payment Posted*, *Result Ready* or *Bed Released*.  Cerner’s Discern Expert rules are triggered by events (e.g., charting a smoker or placing an order) and can evaluate data and take actions like suggesting a PowerPlan【925437338200957†L1967-L1984】. |
| **Timer** | A time‑based trigger that can cause escalations or transitions (e.g., *escalate if specimen not collected within 20 minutes*). |
| **Disposition** | The outcome of a flow, such as *Discharge*, *Admit*, *Transfer* or *Refer*. |
| **Actors** | Roles interacting with the system: Patient and caregiver, Doctor, Nurse, Front Desk, Billing clerk, Laboratory Technologist, Radiographer, Pharmacist, Environmental Services (EVS) staff, Bed/OR manager, Case manager and Quality/Risk management. |
