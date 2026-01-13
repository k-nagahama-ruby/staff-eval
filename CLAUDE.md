## Staff Evaluation App — CLOUDE.md Summary (Next.js + Supabase)

### Goal

Digitize a paper-based staff evaluation process into a small internal web app (≈26 employees), used **twice a year**.
Managers evaluate their team members, employees can view only their own results, the President sees company-wide distributions, and **System Admin** manages configuration and master data.

---

## Core Roles & Permissions

### Roles

- **System Admin**
- **President**
- **Manager**
- **Employee**

---

### Permission Rules (enforced via Supabase RLS)

#### System Admin

- Full access to **system configuration**
- Can:
  - Create / edit / delete **review cycles**
  - Create / edit / delete **evaluation question sets** and questions
  - Manage **organizational data** (departments, manager–employee relationships)
  - View all reviews and answers (read-only by default; write access only if needed)

- Does **not** participate in evaluations as a reviewer by default

#### President

- Read-only access to **all submitted reviews**
- Can view:
  - Company-wide summary dashboards
  - “Distribution for the president” (overall_band, total_score)

- No editing of reviews or questions

#### Manager

- Can:
  - Create and edit reviews **only for their direct reports**
  - Fill in evaluation answers
  - Submit reviews

- Can view:
  - Summary charts for **their own team only**

#### Employee

- Can:
  - View **only their own** reviews

- Cannot:
  - View others’ reviews
  - Edit submitted reviews

---

## MVP Pages

### System Admin

1. **System Settings**
   - Manage review cycles
   - Manage evaluation question sets (versioned)
   - Manage departments and reporting lines (manager_id)

---

### Manager

2. **My Team Reviews**
   - List of reviews for direct reports
   - Filter by:
     - Review cycle
     - Status (draft / submitted)

3. **Review Editor**
   - Load questions from a versioned question set
   - Upsert answers
   - Submit review (locks it)

---

### Employee

4. **My Reviews**
   - View own reviews only
   - Read-only once submitted

---

### President

5. **Summary Dashboard**
   - Company-wide distributions:
     - overall_band
     - total_score

   - Breakdowns:
     - by department
     - by manager
     - by review cycle

   - Dedicated **“distribution for the president”** view

---

## Data Model (Supabase)

### Profiles / Organization

`profiles`

- `id (uuid, pk, auth.users.id)`
- `name`
- `role` (`system_admin | president | manager | employee`)
- `department_id`
- `manager_id (nullable → profiles.id)`

---

### Review Cycles

`review_cycles`

- `id (uuid)`
- `name`
- `start_date`
- `end_date`
- `status` (draft / active / closed)
- `created_at`

---

### Evaluation Questions (Versioned)

`question_sets`

- `id (uuid)`
- `name`
- `version`
- `created_at`

`questions`

- `id (uuid)`
- `question_set_id (fk)`
- `order_index`
- `text`
- `type` (rating / text / etc.)

---

### Reviews

`reviews`

- `id (uuid)`
- `cycle_id`
- `reviewee_id`
- `reviewer_id`
- `question_set_id`
- `status` (draft / submitted / locked)
- `submitted_at`
- `total_score`
- `overall_band`

---

### Answers

`review_answers`

- `id (uuid)`
- `review_id`
- `question_id`
- `value_numeric`
- `value_text`
- `updated_at`

**Unique constraint:** `(review_id, question_id)`
→ Enables upsert

---

## RLS Policy Requirements (Critical)

- **System Admin**
  - Can bypass most restrictions (explicit role check)

- **Reviews**
  - Manager: `reviewer_id = auth.uid()` AND reviewee is direct report
  - Employee: `reviewee_id = auth.uid()`
  - President: read-only, all rows

- **Review Answers**
  - Access controlled via parent review

- **Question Sets / Cycles**
  - Editable only by System Admin

- Prefer **SECURITY DEFINER SQL functions** for:
  - `is_system_admin(uid)`
  - `is_direct_report(manager_uid, employee_uid)`

---

## Status & Lock Rules

- Reviews start as `draft`
- On submit:
  - `status = submitted`
  - `submitted_at = now()`

- Submitted reviews:
  - Read-only in UI
  - RLS blocks updates

---

## Dashboard Strategy

- Store `total_score` and `overall_band` on submit (simplest MVP)
- Dashboards query `reviews` directly
- President dashboard is company-wide
- Manager dashboard is team-scoped

---

## Tech Stack

- **Next.js (React)**
- **Supabase** (Postgres, Auth, RLS)
- Chart library for pie / distribution graphs
- `.env.local` with public Supabase keys

---

## MVP Acceptance Criteria

- System Admin can fully configure cycles and questions
- Managers can evaluate only their team
- Employees can see only their own results
- President can see company-wide distributions
- RLS guarantees zero data leakage even with malicious requests
