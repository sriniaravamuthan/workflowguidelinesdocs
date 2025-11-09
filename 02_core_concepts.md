# Core Concepts

This section elaborates on the building blocks of the workflow‑orchestration layer.  It explains how orders, tasks and instructions behave over time and how they are bundled into reusable order sets.  It also introduces event‑driven triggers and the principles used to connect steps together.

## Order Lifecycle

An **order** represents a clinical request such as a lab test, imaging study, medication or procedure.  Orders progress through distinct states so that downstream systems and staff know what to do next and what has already happened.  The conceptual lifecycle includes the following stages:

1. **Proposed:** The order is part of a pending plan or draft order set.  It can still be edited or removed before being authorized.
2. **Authorized:** A clinician signs the order; it is now a real request in the system.  In Meditech, once an order is selected from a set, it auto‑populates into the order entry screen and can be reviewed or modified before saving【745034776155115†L919-L950】.
3. **Activated:** The order is transmitted to the appropriate department (lab, imaging, pharmacy, etc.).  Meditech shows a “Transmitted” status when the order has been sent to a department【745034776155115†L979-L1016】.
4. **In Progress:** Work on the order has started (specimen collected, imaging performed, medication prepared, etc.).  Statuses such as “In Process” or “Taken” indicate that ancillary staff are working on the order【745034776155115†L979-L1016】.
5. **Resulted / Dispensed / Completed:** The requested service is finished.  Laboratories post results, imaging staff upload reports, pharmacists dispense medications, etc.  The status may be “Complete” or “Resulted”【745034776155115†L979-L1016】.
6. **Verified:** A clinician reviews and accepts the result or verifies that the medication was administered appropriately.  Verification ensures that nothing falls through the cracks before closing the order.
7. **Closed:** The order is fully complete and no further action is needed.
8. **Cancelled:** If an order must be voided (for example, the patient left or there is an allergy conflict), the system triggers compensations to stop further work and reverse any charges.  The Meditech manual notes that orders can be amended or cancelled and warns that cancellations must be done carefully when an order belongs to a series or set【745034776155115†L1041-L1054】.

**Compensations:** At any point until the order is closed, a cancellation routine may be invoked.  Compensation actions might include reversing charges, stopping pharmacy dispensing or notifying lab staff to discard specimens.  These routines must be defined in the workflow designer so that they can be executed automatically when needed.

## Task Lifecycle

A **task** is a human activity requiring attention from a specific role or set of roles.  Tasks can originate from orders (e.g., “Collect specimen”) or from administrative steps (e.g., “Collect copay”, “Schedule imaging”).  They progress through the following states:

1. **Queued:** The task is created and placed into the appropriate inbox or queue.  Sunrise’s Workflow Management tool presents a list of activities and allows clinicians to assign themselves or others【736777591292277†L16-L22】.
2. **Assigned:** A user accepts the task.  Some tasks can be auto‑assigned based on role or availability; others remain unassigned until someone picks them up.
3. **In Progress:** The assignee is actively working on the task.  The system may display a status indicator such as “started but not complete”【736777591292277†L62-L71】.
4. **Completed:** The task is finished and any outputs (forms, documents, results) are recorded.  Completion might trigger the next step in the workflow.
5. **Rejected / Reassigned:** A task can be rejected (e.g., wrong person or missing prerequisites) or reassigned to another role.  Reasons for rejection should be captured for audit and process improvement.
6. **Expired / Escalated:** If a task is not completed within its service‑level agreement, the system escalates it.  For example, a “Collect Copay” task might have an SLA of 10 minutes; if exceeded, the front desk lead is alerted.  Escalations continue until the task is completed or the workflow is cancelled.

Each task has:

* **Assignee roles:** one or more roles eligible to perform the work (e.g., nurse, lab technician, front desk).
* **Inputs:** data fields that must be filled (e.g., specimen barcode, payment amount).
* **Outputs:** results of the task (e.g., specimen collected time, copay collected indicator).
* **SLA/Timer:** expected duration; triggers reminders and escalations if overdue.
* **Escalation policy:** who to notify and when to reassign or cancel.
* **Visibility:** which teams or departments can see the task.

## Instruction Semantics

An **instruction** is a directive or guideline associated with a workflow.  It is not an order but still influences care.  Examples include *“NPO after midnight”*, *“Isolation precautions required”*, or *“Do not discharge until MRI report reviewed.”*

Instructions have two parts:

* **Text with structured tags:** The free text may include tags to aid search, analytics and gating (e.g., `fasting`, `isolation`, `no‑discharge`).  Tags can also support multilingual presentation for patient‑facing instructions.
* **Blocking flag:** A blocking instruction prevents downstream steps until it is acknowledged or cleared.  For example, a discharge step may be blocked until patient education is completed and acknowledged.

Instructions can accompany orders or tasks, and they may be stored in a library for reuse.  They support patient safety and quality initiatives; for instance, decision support systems often deliver guidelines and reminders (documentation templates, order sets and clinical workflow tools) as part of their functions【940019348011578†L276-L279】.

## Order Set Semantics

An **order set** groups together orders, tasks and instructions that are commonly used for a specific condition or procedure.  In Cerner, these are called **PowerPlans**; they allow clinicians to define collections of clinical content that expedite multiple steps of an intervention【925437338200957†L1935-L1944】.  Meditech describes an order set as a group of tests or procedures; selecting an order set automatically populates all of its items into the order entry screen and allows clinicians to edit or remove orders before saving【745034776155115†L919-L950】.

Order sets in the orchestration layer support the following features:

* **Parallel and sequential groups:** Items within a set can run in parallel (e.g., lab and imaging orders) or sequentially with dependencies.
* **Conditional activation:** A rule can control whether a branch runs.  For example, an imaging order may be activated only if neurological deficits are recorded in the encounter note.
* **Variants:** Clinicians can choose among variants (adult/pediatric dosing, low/high risk protocols).  When applying an order set, they select the appropriate variant, and the underlying orders and tasks adjust accordingly.
* **Templates and favorites:** Doctors can author order sets and save them as personal, team, department or hospital‑wide templates.  Cerner PowerPlans can be built for inpatient or outpatient care and can include immunizations, diagnoses, dispositions and follow‑up information【925437338200957†L1956-L1966】.  Meditech allows default responses to auto‑populate but still gives the user control to change or delete items【745034776155115†L919-L950】.

## Events & Triggers

**Events** are signals—either from the HIMS or external devices—that drive the progression of workflow instances.  They may be:

* **System events:** Appointment booked, patient arrived, payment posted, order created, result ready, discharge ordered, bed released.
* **IoT events:** Signals from monitors (e.g., SpO₂ drop), fridge temperature sensors, smart beds or ambulance GPS updates.  These events can create urgent tasks but should not automate irreversible clinical actions without human confirmation.
* **Manual signals:** Operators may manually send signals such as *Override*, *Hold*, or *Ready to Perform* when circumstances change.

The orchestration engine uses events in combination with rules to decide which steps to trigger.  Cerner’s **Discern Expert** uses an “event → logic → action” pattern: an event (e.g., charting a patient as a current smoker) evokes the rule engine, which evaluates patient data (age, gender, location) and then performs an action, such as suggesting a PowerPlan or placing a referral order【925437338200957†L1967-L1984】.  Discern Expert distinguishes between synchronous events (alerts presented immediately during order entry) and asynchronous events processed in the background; asynchronous events continue to run even after the user closes the chart【925437338200957†L2043-L2077】.  Our system adopts a similar model: waits may listen for synchronous events (e.g., lab result posted) or asynchronous events (e.g., payment confirmation) and time out if the expected event does not arrive.

## Step‑to‑Step Flow Principles

Workflows consist of **steps** connected by transitions.  Each step declares:

* **Preconditions:** Data or events that must be present before the step executes (e.g., patient must have arrived; lab order must be authorized).
* **Inputs:** Context passed into the step (patient demographics, order parameters, previous results).
* **Effects:** Actions taken by the step (create orders, launch tasks, send notifications).  Effects must be idempotent; repeating a trigger should not duplicate charges or bookings.
* **Compensation:** A routine to undo side effects if the step fails or is cancelled.
* **On success:** The next step(s) to run when the step completes successfully.
* **On failure:** Alternative transitions when the step fails (e.g., send to intervention queue, notify supervisor).
* **On timeout:** Behavior if the step does not finish within its SLA (e.g., escalation, cancellation).

**Routing rules** determine which path to take based on data, results, time or manual signals.  These rules may be defined using IF‑THEN expressions similar to knowledge‑based clinical decision support, which rely on retrieving data and producing an action【940019348011578†L228-L260】.  For example, if a lactate level is above a threshold, the rule may add an “Intensivist consult” task.

Workflows often require **parallelism and synchronization**.  Steps may fan out to run lab and imaging subflows concurrently and then join at a checkpoint.  Waits specify which events must occur before progressing (any vs. all) and define what happens when timeouts occur.  **Gates** or **checkpoints** are named conditions that must be satisfied to proceed; for instance, a *Pre‑operative Checklist* gate may require completion of consent, site marking and anesthesia assessment.

By following these principles, the orchestration layer provides a deterministic, auditable and safe framework for coordinating complex patient journeys across multiple systems and roles.
