# Staff Evaluation App — CLAUDE.md

## Overview

Digitize TWM's paper-based employee evaluation process into a secure internal web app (~26 employees). Used **twice a year** for performance reviews.

**Tech Stack:** Next.js (React) + Supabase (Postgres, Auth, RLS)

---

## Core Roles & Permissions

| Role | Capabilities |
|------|-------------|
| **System Admin** | Full system config, manage cycles/templates/users, view all data |
| **President** | Read-only access to all reviews, company-wide analytics |
| **Manager** | Create/edit reviews for direct reports only, view team summaries |
| **Employee** | View own reviews only (read-only after submission) |

---

## Data Model (Supabase)

### Tables

```
profiles
├── id (uuid, pk, = auth.users.id)
├── name (text)
├── role (text: system_admin | president | manager | employee)
├── department (text)
├── manager_id (uuid, nullable → profiles.id)
└── created_at (timestamptz)

review_cycles
├── id (uuid)
├── name (text)
├── start_date (date)
├── end_date (date)
├── status (text: draft | active | closed)
└── created_at (timestamptz)

question_templates
├── id (uuid)
├── version (int4)
├── section (text)
├── prompt (text)
├── type (text: rating | text | choice)
├── is_required (bool)
├── sort_order (int4)
└── created_at (timestamptz)

reviews
├── id (uuid)
├── employee_id (uuid → profiles.id)
├── manager_id (uuid → profiles.id)
├── cycle_id (uuid → review_cycles.id)
├── template_version (uuid)
├── status (text: draft | submitted | locked)
├── total_score (int4)
├── overall_band (text: below | meets | exceeds)
├── created_at (timestamptz)
├── submitted_at (timestamptz)
└── locked_at (timestamptz)

review_answers
├── id (uuid)
├── review_id (uuid → reviews.id)
├── question_id (uuid → question_templates.id)
├── rating_value (int4, 1-5)
├── choice_value (text: exceeded | achieved | not_achieved)
└── text_value (text)
```

**Unique constraint:** `review_answers(review_id, question_id)` → enables upsert

---

## Evaluation Form Structure

### 1. Performance Rating Scale (3-Point)
- **Exceeded** - Performance far exceeded expectations
- **Achieved** - Performance consistently met expectations
- **Not Achieved** - Performance did not consistently meet expectations

### 2. Scale Questions (1-5)
15 questions rated 1 (Poor) to 5 (Excellent):
- Stress handling, deadlines, communication, adaptability
- Problem-solving, leadership, initiative, workload management
- Feedback reception, cross-team communication, time management
- Flexibility, ethical behavior, going above and beyond

### 3. Open-Ended Questions
20 text questions covering:
- Strengths and achievements
- Areas for improvement
- Team contribution and collaboration
- Growth and development needs

### 4. Conclusion Ratings
- Core Competencies & Values (3-point)
- Employee Strengths & Achievements (3-point)
- Areas of Improvement (text)
- Manager's Comments (text)
- Future Goals (text)

### 5. Overall Rating (Auto-calculated)
| Score Range | Rating |
|-------------|--------|
| ≤30 points | Below Expectations |
| 45-50 points | Meets Expectations |
| >60 points | Exceeds Expectations |
| 31-44, 51-60 | Unmapped (admin must configure) |

---

## RLS Policy Requirements

### Security Functions (SECURITY DEFINER)
```sql
is_system_admin(uid) → boolean
is_direct_report(manager_uid, employee_uid) → boolean
```

### Access Rules

**reviews table:**
- System Admin: full access
- Manager: `reviewer_id = auth.uid()` AND reviewee is direct report
- Employee: `reviewee_id = auth.uid()` (read-only)
- President: read-only, all rows

**review_answers table:**
- Access controlled via parent review relationship

**question_templates / review_cycles:**
- Editable only by System Admin
- Read access for all authenticated users

---

## Status & Lock Rules

1. Reviews start as `draft`
2. On submit:
   - `status = 'submitted'`
   - `submitted_at = now()`
   - `total_score` and `overall_band` computed and stored
3. Submitted reviews:
   - Read-only in UI
   - RLS blocks updates (except admin reopen)

---

## MVP Pages

### System Admin
- **System Settings** - Manage cycles, question templates (versioned), departments, reporting lines

### Manager
- **My Team Reviews** - List reviews for direct reports, filter by cycle/status
- **Review Editor** - Load questions, fill answers, submit (locks review)

### Employee
- **My Reviews** - View own reviews only (read-only after submission)

### President
- **Summary Dashboard** - Company-wide distributions (overall_band, total_score), breakdowns by department/manager/cycle

---

## Dashboard Strategy

- Store `total_score` and `overall_band` on submit
- Query `reviews` table directly for dashboards
- President dashboard: company-wide scope
- Manager dashboard: team-scoped (direct reports only)
- Charts: distribution histograms, pie/donut charts

---

## MVP Acceptance Criteria

1. System Admin can fully configure cycles and question templates
2. Managers can evaluate only their direct reports
3. Employees can see only their own results
4. President can see company-wide distributions
5. RLS guarantees zero data leakage
6. Total score and overall rating computed on submit
7. Template versioning works for future cycles

---

## Key Files Reference

- `er-diagram.png` - Database ER diagram
- `requirement-define.md` - Full requirement specification
- `employee-evaluation-form.md` - Actual question templates from paper form

---

## Development Notes

### Scoring Validation
- Scale questions: reject values outside 1-5
- Show missing questions on submit attempt
- Handle score gaps (31-44, 51-60) with admin-configurable thresholds

### Change Control
- Active cycles: restrict template changes
- Allow only non-breaking edits (label/help text) once submissions exist

### Privacy
- Open-ended text contains sensitive feedback
- Enforce "least privilege" access
- Track audit events: review_created, review_updated, review_submitted, etc.
