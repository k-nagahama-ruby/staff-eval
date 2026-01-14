## 1. Purpose and goals

### Purpose

Digitize TWM’s paper-based employee evaluation process into a secure web app that supports:

* Creating and distributing evaluation forms
* Collecting quantitative (1–5) and qualitative feedback
* Manager evaluation + overall rating
* Company-level insights for leadership (including charts and distributions)

### Success criteria

* Managers can complete reviews for their team and see summaries easily.
* Employees cannot see others’ reviews; access is strictly role-based.
* Admin can edit evaluation questions without code changes.
* Leadership (President) can view organization-wide distributions.

---

## 2. Stakeholders & user roles

### Roles

1. **System Admin**

   * Manages evaluation templates/questions, cycles, and system settings
2. **Manager**

   * Creates and completes reviews for direct reports
   * Views team summary dashboards
3. **Employee**

   * Can view *only* their own review results (if enabled by policy)
   * Cannot see any other employee’s reviews
4. **President**

   * Views company-wide analytics and distributions (read-only)
   * May view per-department/per-manager rollups (optional)

> Note: You can implement President as a separate role or as a privileged Manager/Executive role with additional permissions.

---

## 3. Evaluation form content (from current paper form)

look at **employee-evaluation-form.md**
---

## 4. Scoring and rating rules

### 4.1 Scale question scoring

* Each scale question is answered with an integer **1–5**.
* Total score = sum of all scale questions answered.

**Validation**

* Must reject values outside 1–5.
* Must show which questions are missing if submission attempted.

### 4.2 Overall rating thresholds (as given)

Overall rating is determined by total points:

* **Below expectations:** **30 points and below**
* **Meets expectations:** **45 to 50 points**
* **Exceeds expectations:** **above 60 points**


**Important requirement (gap handling)**
The thresholds leave gaps (e.g., 31–44, 51–60). The system must support one of these approaches:

**Recommended default behavior (configurable):**

* Thresholds are **admin-configurable** per evaluation cycle/template.
* If a score falls into an undefined range, the UI shows:

  * “Unmapped score range — please adjust thresholds” to Admin/Manager
  * And prevents finalization until resolved (to avoid ambiguous outcomes)

### 4.3 3-point competency ratings

For “Core competencies and core values” and “Employee strengths, achievements, and contributions”:

* User selects exactly one of: **Exceeded / Achieved / Not Achieved**.
* These are independent from numeric total score, but appear in final summary.

### 4.4 Normalization rule for duplicated/overlapping items

Some items may overlap (e.g., “problem-solving” appears multiple times). The system must:

* Treat each question as a unique question ID in the template.
* Admin can merge/disable duplicates in the template editor for future cycles.

---

## 5. Core workflows

## 5.1 Setup workflow (System Admin)

1. Create an **Evaluation Cycle**

   * Name (e.g., “2026 H1 Review”)
   * Start date / end date
   * Status: Draft → Active → Closed → Archived
2. Choose or create an **Evaluation Template** for that cycle
3. Define:

   * Scale questions (1–5)
   * Open-ended questions
   * Conclusion section fields
   * Scoring thresholds (configurable)
4. Assign participating employees and their managers (import or sync)
5. Activate the cycle

**Acceptance criteria**

* Admin can preview the entire form exactly as reviewers will see it.
* Template changes in an Active cycle must be restricted (see change control).

---

## 5.2 Review creation workflow (Manager)

1. Manager selects an active cycle
2. Manager sees a list of direct reports requiring review
3. For each employee, manager creates a **Review**

   * Pre-fills: employee name, department, manager name, review date
4. Manager completes:

   * Scale questions (1–5)
   * Open-ended responses
   * Competency rating (3-point)
   * Strengths/achievement rating (3-point)
   * Areas of improvement, manager comments, future goals
5. Submit review (or save draft)

**Acceptance criteria**

* Draft saves partial progress.
* Submit validates required fields.
* After submit, review becomes locked (editable only by Admin, or via “reopen” action).

---

## 5.3 Employee access workflow

Policy must be configurable per cycle:

*  Employee can view finalized review after cycle is closed

**Acceptance criteria**

* Employees can only access their own reviews.
* No employee can see peer reviews or company distribution.

---

## 5.4 President (Leadership) analytics workflow

President can view:

* Company-wide distribution of scores and overall ratings
* Breakdown by department, manager, cycle
* Ability to drill into aggregated views, not necessarily individual text answers (configurable)

**Acceptance criteria**

* President dashboard includes **distribution visualization** (histogram/bar) and **circle graph** (e.g., pie/donut).
* Access is read-only.

---

## 6. Functional requirements by module

## 6.1 Authentication & user management

* Login via email/password
* Users have:

  * Name
  * Email
  * Department
  * Role(s)
  * Manager relationship (manager_id)

**Admin capabilities**

* Create/edit users
* Bulk import users (CSV)
* Assign managers and departments

---

## 6.2 Evaluation cycles

* CRUD cycles (Admin)
* Cycle status controls feature availability:

  * Draft: editable setup only
  * Active: managers can create/submit reviews
  * Closed: no new submissions; viewing depends on policy
  * Archived: read-only

---

## 6.3 Templates & question editor (Admin)

Template must support:

* Sections
* Question types:

  * Scale (1–5)
  * Long text
  * Single-choice (Exceeded/Achieved/Not Achieved)
* Required flag per question
* Ordering
* Help text
* Ability to add/remove/disable questions
* Versioning strategy:

  * A template can be reused across cycles
  * A cycle “locks” a template version once activated

**Change control requirement**

* If cycle is Active and template changes are made:

  * Either disallow changes, OR
  * Allow only non-breaking edits (label/help text), and prevent changing scoring logic once submissions exist

---

## 6.4 Review entry

* Manager fills out form for each employee
* Autosave draft (recommended)
* Field validation
* Submission creates immutable snapshot of:

  * Template version
  * All responses
  * Score calculation result
  * Overall rating result

---

## 6.5 Scoring engine

* Calculate totals for scale questions
* Determine overall rating based on configured thresholds
* Store:

  * raw_total_score
  * max_possible_score (based on number of scale questions * 5)
  * normalized_percent (optional)
  * overall_rating (Below/Meets/Exceeds/Unmapped)

---

## 6.6 Dashboards & reports

### Manager dashboard

* List of direct reports
* Completion progress for the cycle
* Team summary:

  * average score
  * distribution chart (team-level)
  * list of “submitted vs pending”

### President dashboard

* Company distribution chart
* Breakdown filters:

  * Cycle
  * Department
  * Manager
* Export aggregated results (CSV)

### Admin dashboard

* Cycle status overview
* Submission progress
* Template management shortcut
* Audit logs overview

---

## 6.7 Exporting

Must support at least:

* Export a single employee review to PDF (optional but highly useful)
* Export cycle results to CSV:

  * per review: employee, department, manager, total score, overall rating, competency ratings, timestamps
* Export should respect permissions (e.g., managers only their team)

---

## 6.8 Notifications (optional)

* When cycle opens: notify managers
* Reminder: pending reviews
* When review submitted: optional notify employee (if employee-view enabled)

(Email delivery can be deferred to Phase 2.)

---

## 7. Permissions & security (RLS-ready)

### Data access rules

* **Employee**

  * Read: own review(s) only (and only when permitted by cycle policy)
  * No access to other employees’ reviews
* **Manager**

  * CRUD drafts for direct reports
  * Submit reviews for direct reports
  * Read team summaries and finalized reviews for direct reports
* **President**

  * Read: all-cycle aggregated analytics
  * Optional read: individual review summaries (configurable)
* **Admin**

  * Full access

### Audit log requirements

Track events:

* review_created, review_updated, review_submitted, review_reopened
* template_updated, cycle_activated, cycle_closed
  Store: actor_user_id, timestamp, entity_type, entity_id, diff summary (optional)

### Privacy

* Open-ended text can contain sensitive feedback; ensure strict access.
* “Least privilege” is required.

---

## 8. Data model

look at **er-diagram.png**

## 9. UI / Pages

### Common

* Login
* Role-based navigation

### Admin

* Users management
* Evaluation cycles list + detail
* Template editor (with preview)
* Cycle analytics + progress

### Manager

* “My team” list (per cycle)
* Review form (draft/submit)
* Team summary dashboard (charts)

### Employee

* “My reviews” list (if enabled)
* Review detail (read-only)

### President

* Company analytics dashboard

  * Filters: cycle / department / manager
  * Charts: donut/pie + distribution graph


## 10. Acceptance criteria (MVP)

1. Admin can create a cycle and configure a template with:

   * scale questions (1–5)
   * open-ended questions
   * competency ratings (3-point)
   * thresholds
2. Manager can create, save draft, and submit a review for each direct report.
3. Total score and overall rating are computed and stored on submit.
4. Manager sees team progress + summary charts.
5. President sees company-wide distribution and can filter by cycle/department.
6. Employees cannot access others’ reviews (verified by RLS).
7. Admin can edit questions for future cycles (template versioning works).

