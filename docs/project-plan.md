# Pulso: Project Plan, Design Guide & Documentation Guide

**Event:** HackathON (LSPU-SPCC, organized by OCTA) · **Theme:** Student Around Services **Tracks:** EdTech/Academic Accessibility · Data-Driven Administration · Quality of Life **Deadline:** Nov 12, 12 NN (code freeze and submission) · Pitching at 1 PM

> **Pitch hook:** Students don't fail because of one hard subject. They fail because of week 8. Pulso makes week 8 visible before it happens.

---

> **Identity decision:** University Google accounts must use the `0000-0000@lspu.edu.ph` format. An approved administrator assigns application roles and section memberships; sign-in alone does not grant access.

## 1. Overview

### 1.1 What Pulso is

Pulso is a workload early-warning system. Instructors enter assessment deadlines for their sections, and Pulso builds a deadline timeline for every enrolled student, scores how heavy each week is, and warns **instructors before they post a deadline that collides** with their students' other work.

### 1.2 The problem

- Each instructor sees only their own subject, so a deadline looks reasonable in isolation.
- Each student discovers the pile-up only when it arrives.
- Administrators have no data to justify staggering exams or project dates.

### 1.3 The key insight

Most student-wellness tools help students *cope* with workload. Pulso prevents the pile-up at the *source* by giving the people who set deadlines the same shared view that students suffer from.

### 1.4 Users and what they get

| User | Gets | Why they use it |
| --- | --- | --- |
| Student | Personal timeline, weekly load meter, AI-assisted study plan | See heavy weeks early and get a practical plan for handling them |
| Instructor | Manual deadline management for their sections, plus a collision warning when posting a deadline and lighter date suggestions | Avoid being the last straw without coordinating with every colleague |
| Dept head / admin | Anonymized heatmap of crunch weeks by program and year level | Evidence to stagger midterms and project due dates |

---

## 2. Features

### 2.1 Instructor deadline management

Instructors manually add, edit, and delete assessment deadlines for the sections they handle. Each deadline records the section, title, assessment type, exact due date, and grade weight (when known). Once saved, the deadline automatically appears on the timeline of every student enrolled in that section.

There is **no syllabus upload, file import, document parsing, extraction, or extraction-review screen** in Pulso. The instructor enters the deadline details directly through the application.

Example deadline record:

```json
{
  "section_id": "section_302",
  "subject": "CSEL 302",
  "title": "Midterm Exam",
  "type": "exam",
  "due_date": "2026-12-10",
  "weight_percent": 25
}
```

**Rationale:** direct entry removes the document-processing workflow and gives Pulso one clear source of truth: deadlines intentionally entered and confirmed by the instructor responsible for the section.

### 2.2 Personal timeline and load meter (student)

A calendar with a color-coded weekly load bar (green, yellow, red). Clicking a week lists everything due and shows the score breakdown.

### 2.3 Faculty collision check (instructor): the differentiator

When an instructor adds a deadline for a section, Pulso computes each enrolled student's load for that week and responds, for example:

> 12 of 40 students already have 2+ major deadlines this week. Thursday, Dec 10 would reduce this to 3.

It shows the 2-3 best alternative dates. Pulso **advises and never blocks**; the instructor can ignore it.

### 2.4 AI study assistant and plan generator (student)

When a student opens a heavy week, they can ask Pulso to generate a practical study plan. The LLM receives only the deadlines already visible to that student, their due dates and assessment types, the calculated workload breakdown, and the student's available study days and maximum study time per day for the planning period. It returns a structured plan with suggested study tasks, dates, time estimates, and brief explanations.

The student reviews the suggestions and can adjust them. The AI plan is advisory: it never changes an official deadline, edits a section's records, or overrides the load score. The backend validates every suggested task and date against the supplied deadlines and availability. If the model fails or returns invalid output, Pulso falls back to a simple rule-based plan.

**Rationale:** this is where an LLM adds value: turning confirmed workload data into understandable, personalized guidance. The workload score and scheduling constraints remain deterministic and testable.

### 2.5 Admin heatmap

A grid with weeks on one axis and program/year level on the other, colored by average load. Data is aggregated and anonymized.

### 2.6 Priority and cut line

| Priority | Feature | Note |
| --- | --- | --- |
| 1 | Deadline model and manual CRUD, student timeline, load score | Core, build first |
| 2 | Faculty collision check | The differentiator; keep calculations deterministic |
| 3 | AI study assistant and structured study plan | Main LLM feature and student-facing demo moment |
| 4 | Admin heatmap on seeded data | Great for the demo |
| 5 | Advanced conversational features and extensive personalization | Cut these before the core AI plan if time is limited |

---

## 3. The Load Score

The score is the heart of Pulso. It is intentionally simple so it can be explained on one slide.

**Rationale:** weight each item by effort (type) and by how much it counts toward the grade, then sum per week. Fixed weights keep it transparent and defensible. Judges can see exactly why a week is red.

```typescript
const TYPE_WEIGHT = { quiz: 1, activity: 2, exam: 3, performance: 4 } as const;

type Item = { type: keyof typeof TYPE_WEIGHT; weightPercent: number };

export function itemLoad(item: Item): number {
  // Grade weight scales base effort: a 30% project counts more than a 5% one
  return TYPE_WEIGHT[item.type] * (1 + item.weightPercent / 100);
}

export function weeklyLoad(itemsInWeek: Item[]): number {
  return itemsInWeek.reduce((sum, i) => sum + itemLoad(i), 0);
}

export function band(score: number): "green" | "yellow" | "red" {
  if (score < 6) return "green";
  if (score < 10) return "yellow";
  return "red";
}
```

**Calibration:** the thresholds (6 and 10) are placeholders. After generating seed data, adjust them so roughly 15-20% of student-weeks come out red.

### 3.1 Collision suggestion logic

```typescript
const DAY_MS = 24 * 60 * 60 * 1000;

// averageSectionLoad, weekOf and newItem are helpers you write against your Convex data
export function suggestDates(
  sectionId: string,
  intendedDate: Date,
  windowDays = 7,
  topN = 3
): Date[] {
  const candidates: { avg: number; date: Date }[] = [];
  for (let offset = -windowDays; offset <= windowDays; offset++) {
    const d = new Date(intendedDate.getTime() + offset * DAY_MS);
    const day = d.getDay();
    if (day === 0 || day === 6) continue; // skip weekends
    const avg = averageSectionLoad(sectionId, weekOf(d), newItem(d));
    candidates.push({ avg, date: d });
  }
  candidates.sort((a, b) => a.avg - b.avg);
  return candidates.slice(0, topN).map((c) => c.date);
}
```

The function recomputes the section's average weekly load as if the new deadline sat on each candidate date and returns the three lowest.

---

## 4. System Design

### 4.1 Architecture

```
[ Next.js / React app on Vercel ]  <->  [ Convex ]
                                          |- queries / mutations: load score, collision check, heatmap
                                          |- database + auth
                                          '- actions --> [ Gemini API / Mistral fallback ]  (AI study-plan generation)
```

- **Frontend:** a React-based app (Next.js or Vite) hosted on Vercel. A calendar library saves a lot of time.
- **Backend and DB:** Convex provides the database, auth, and server logic in TypeScript. The load score, collision check, and heatmap aggregation are queries and mutations, so no separate server is needed.
- **AI study planning:** a Convex action sends a student's confirmed deadline data, calculated workload breakdown, and planning availability to Gemini using a strict structured-output prompt. The response is validated in code. If Gemini is rate-limited or fails, the action retries with Mistral's free tier; if both fail, Pulso uses its rule-based fallback. Both keys are stored as Convex environment variables.
- **Deploy:** Vercel for the frontend and Convex for the backend, so judges get a live link.

> **Check first:** the orientation deck lists Tech Stack and Platform Usage rules without details. Confirm with the organizers that an LLM API is allowed.

### 4.2 Data model

```sql
CREATE TABLE users (
  id          SERIAL PRIMARY KEY,
  name        TEXT NOT NULL,
  role        TEXT CHECK (role IN ('student','instructor','admin')),
  program     TEXT,
  year_level  INT
);

CREATE TABLE sections (
  id            SERIAL PRIMARY KEY,
  subject       TEXT NOT NULL,
  instructor_id INT REFERENCES users(id)
);

CREATE TABLE enrollments (
  student_id INT REFERENCES users(id),
  section_id INT REFERENCES sections(id),
  PRIMARY KEY (student_id, section_id)
);

CREATE TABLE deadlines (
  id             SERIAL PRIMARY KEY,
  section_id     INT REFERENCES sections(id),
  title          TEXT NOT NULL,
  type           TEXT CHECK (type IN ('quiz','activity','exam','performance')),
  due_date       DATE NOT NULL,
  weight_percent INT DEFAULT 0,
  created_by     INT REFERENCES users(id) NOT NULL
);
```

**Rationale:** weekly load is computed by joining instructor-entered `deadlines` with `enrollments`, so no separate load table is needed. This avoids stale data. `created_by` records which instructor entered the deadline; there is no syllabus/import source field.

### 4.3 Convex function outline

| Function | Type | Purpose |
| --- | --- | --- |
| `deadlines.create` | mutation | Create one instructor-entered deadline for a section |
| `deadlines.update` | mutation | Edit an existing section deadline |
| `deadlines.remove` | mutation | Delete a deadline after permission checks |
| `students.timeline` | query | Confirmed deadlines plus weekly load bands and score breakdowns |
| `deadlines.collisionCheck` | query | Section, date, type, weight in; impact and 3 suggestions out |
| `students.generateStudyPlan` | action | Send the student's visible deadlines, workload context, and availability to Gemini (Mistral fallback); validate structured plan output |
| `admin.heatmap` | query | Aggregated load by week and program/year |

### 4.4 AI study-plan prompt (starting point)

```text
You are Pulso's study-planning assistant. Create a realistic study plan using
ONLY the supplied confirmed deadlines, workload breakdown, planning period,
and the student's available study days and time limits.

Return ONLY valid JSON. Do not include markdown fences or prose outside JSON.
Use this shape:
{
  "summary": "short explanation of the week's workload and approach",
  "schedule": [
    {
      "date": "YYYY-MM-DD",
      "deadline_id": "an ID from the supplied deadline list",
      "task": "a specific, practical study action",
      "minutes": 45,
      "reason": "brief reason for placing this task on this date"
    }
  ],
  "tips": ["short, practical suggestion"]
}

Rules:
- Never create, rename, or assume a deadline that is not in the input.
- Every deadline_id must match an item supplied in the input.
- Schedule work only within the planning period and on dates no later than
  that deadline's due date. Prefer starting before the due date.
- Respect the student's available days and maximum study time per day.
- Do not change due dates, assessment types, grade weights, or workload scores.
- Do not invent assignment requirements. Keep task suggestions general when
  the deadline record does not include task instructions.
- Use the supplied workload breakdown as context; do not recalculate it.
- Return an empty schedule with a clear summary when the supplied constraints
  do not allow a valid plan. Do not silently break the constraints.
```

The backend must validate the JSON shape, deadline IDs, planning dates, due-date limits, and daily minute limits before showing the plan. The prompt is a starting point; enforce constraints in code rather than relying on prompt instructions alone.

### 4.5 Seed data

Generate about 100 fake students across 4-6 sections and 3 programs with a script. Include deliberate crunch weeks (for example, week 8 and week 15) so the heatmap and collision check show a clear story.

### 4.6 How the LLM is used

The LLM is used for the **student-facing study assistant**, not for collecting or deciding official academic deadlines. Instructors manually enter and confirm deadlines in Pulso. The application remains usable when the LLM is unavailable.

1. **Trigger:** a student selects a week or planning period from their timeline and chooses **Generate study plan**.
2. **Gather context in code:** Convex retrieves only the deadlines that the signed-in student is allowed to see, along with the calculated weekly load, the individual items contributing to that score, and the planning-period boundaries. The student supplies available study days and a daily time limit.
3. **Prepare a minimal request:** the backend sends the model structured deadline data and workload context. Do not send the student's name, email, account ID, unrelated course records, or raw syllabus/document files.
4. **Generate:** a Convex action sends the request to Gemini with the structured-output prompt. If Gemini is rate-limited or fails, it retries with Mistral.
5. **Validate in code:** check that every referenced deadline exists in the supplied context, all scheduled dates are within the planning period and not after the corresponding due date, and each day's total minutes stays within the student's availability. Reject invalid output; retry once or use the deterministic fallback.
6. **Review:** show the suggested plan as editable advice. The student can adjust or ignore it. Generating a plan never changes official deadlines, instructor records, load scores, or collision results.

**What the LLM does:** organize work into understandable tasks, explain why a week may feel heavy using the supplied breakdown, and tailor suggestions to the student's stated availability.

**What the LLM does not do:** extract deadlines, invent missing assessments, calculate workload scores, decide whether a deadline collision exists, choose official alternative dates, or write to the database. Those responsibilities stay in deterministic application code.

**Fallback:** if both model providers fail or the response fails validation, generate a basic plan in code using the confirmed deadlines, due dates, and available study windows. The student should still be able to use the timeline, load meter, and collision check without any AI response.

**Rationale:** this places AI where natural-language guidance and personalization are useful while keeping academic records and calculations explainable. Since the model receives only a student's relevant deadline context, availability, and workload breakdown, the prompt is smaller and avoids unnecessary personal data.

---

## 5. Recommended Design

### 5.1 Design principles

1. **Glanceable first.** The weekly color band must be readable in under two seconds.
2. **Calm, not alarming.** Red means attention, not panic. Use soft tones and clear language.
3. **Advise, never block.** Every warning has a dismiss or ignore path.
4. **One screen, one job.** Student, instructor, and admin each get a focused home screen.
5. **Never rely on color alone.** Pair each band with an icon and label.

### 5.2 Color palette

| Role | Color | Hex |
| --- | --- | --- |
| Primary (brand) | Indigo | `#4F46E5` |
| Background | Soft slate | `#F8FAFC` |
| Surface (cards) | White | `#FFFFFF` |
| Text | Ink navy | `#0F172A` |
| Muted text | Slate | `#64748B` |
| Border | Light slate | `#E2E8F0` |
| Load: Green (light) | Green | `#16A34A` |
| Load: Yellow (caution) | Amber | `#F59E0B` |
| Load: Red (critical) | Red | `#DC2626` |

Provide a dark theme by swapping background to `#0B1220` and surface to `#111A2E`, keeping the three load colors.

### 5.3 Typography

- **Headings:** Plus Jakarta Sans (semi-bold)
- **Body and UI:** Inter
- **Numbers (scores, counts):** Inter with tabular figures
- Scale: 14 / 16 / 20 / 28 / 36 px. Body never below 14 px.

### 5.4 Layout and key screens

**Student home**

```
+--------------------------------------------------+
| Pulso            Week 8 · Dec 1-7                |
+--------------------------------------------------+
| Load this week:  [##########--]  RED  ⚠ Critical |
| 3 exams · 2 projects                             |
+--------------------------------------------------+
| Calendar / timeline (color-coded weeks)          |
+--------------------------------------------------+
| Study availability: days + daily time limit      |
| [Generate AI study plan]                          |
+--------------------------------------------------+
| AI study plan for this week                       |
| Suggested tasks grouped by day                   |
+--------------------------------------------------+
```

**Instructor: add deadline**

```
+--------------------------------------------------+
| New deadline · CSEL 302 · Dec 3                  |
+--------------------------------------------------+
| ⚠ 12 of 40 students already have 2+ deadlines    |
| Lighter options:  Dec 10 (3)   Dec 8 (5)  Dec 11 |
| [Use suggestion]            [Keep my date]       |
+--------------------------------------------------+
```

**Admin heatmap**

```
            W1  W2  W3 ... W8  W9 ... W15
BSIT-1      🟩  🟩  🟨      🟥  🟨      🟥
BSIT-2      🟩  🟨  🟩      🟥  🟥      🟨
BSCS-3      🟩  🟩  🟩      🟨  🟥      🟥
```

### 5.5 Key components

- **Load meter:** horizontal bar with the band color, an icon, and a text label (Light, Moderate, Critical).
- **Week card:** week range, band, list of items, score breakdown on expand.
- **Collision banner:** inline card with impact numbers and two actions: use a suggestion or keep my date.
- **Heatmap cell:** colored square with a tooltip showing average load and student count.
- **AI study-plan card:** generated schedule grouped by date, with task duration and a short reason; clearly marked as a suggestion and editable by the student.

### 5.6 Accessibility and usability

- Contrast ratio of at least 4.5:1 for text.
- Load bands use icon plus label, not color alone (for color-blind users).
- Full keyboard navigation and visible focus states.
- Touch targets of at least 44 px, since many students will use phones.
- Mobile-first layout for the student views; admin views may be desktop-first.

### 5.7 Tone of copy

Friendly, direct, and non-judgmental. Prefer "Week 8 looks heavy. Start the project early." over "WARNING: overload."

---

## 6. Build Plan (≈2.5 days)

| When | Goal | Deliverables |
| --- | --- | --- |
| Nov 9 evening | Setup | Repo, stack decided, DB schema, auth, seed script started, design tokens |
| Nov 10 | Core | Manual deadline CRUD, student timeline, load score, seeded data |
| Nov 11 | Differentiator | Collision check and suggestions, AI study-plan action and UI, admin heatmap, seeded data tuned |
| Nov 12 morning | Ship | Deploy, bug fixes, README, deck, demo rehearsal. Code freeze at 12 NN |

### 6.1 Suggested team roles (3-5 members)

- **Backend and data:** schema, API, seed script, collision logic.
- **Pulso engine:** load score, collision/date-suggestion logic, AI study-plan action, and deterministic fallback.
- **Frontend A:** student views.
- **Frontend B:** instructor and admin views, heatmap.
- **Pitch and docs (can be shared):** README, deck, demo script.

### 6.2 Risks and mitigations

| Risk | Mitigation |
| --- | --- |
| LLM returns generic or invalid plans | Use strict structured output, validate every task/date/time limit in code, and provide a deterministic fallback |
| Privacy concerns | Send only the student's relevant deadline context and availability; aggregate admin data and show instructors counts only, never names |
| Incomplete deadline data | Make the instructor responsible for entering and maintaining section deadlines; show the last-updated date where useful |
| "Would instructors use it?" | Position collision feedback as advisory; show how it helps instructors avoid overloading their own students |
| Live demo failure | Use seeded deadlines and a pre-generated fallback study plan; keep a recorded backup |
| Time overrun | Follow the cut line in section 2.6 |

---

## 7. Pitch & Demo

### 7.1 Five-minute demo flow

1. **Hook (30 s):** tell the "week 8" story.
2. **Instructor and student (60 s):** the instructor manually adds a deadline to a section; switch to an enrolled student account, see the deadline and load update, then generate an AI study plan for the heavy week.
3. **Instructor (90 s):** add a deadline, see the collision warning, and pick a lighter date.
4. **Admin (60 s):** show the heatmap and the crunch weeks.
5. **Impact and close (60 s):** how it scales and the closing line.

### 7.2 Mapping to the judging criteria

| Criterion | Weight | How Pulso scores |
| --- | --- | --- |
| Innovation and originality | 35% | Faculty-side collision warning prevents the pile-up at the source |
| Technical complexity and execution | 20% | AI-generated study planning with code validation, transparent load scoring, date suggestion, aggregation |
| Practicality and impact | 20% | Every student has lived week 8 |
| UI/UX and design | 15% | Glanceable meter and heatmap, accessible design |
| Presentation and pitch | 10% | A single, relatable story |

---

## 8. Documentation Guide

The submission requires a **public GitHub repository** and a **PowerPoint deck**, both posted in the Discord submission channel before Nov 12, 12 NN. Good documentation is part of how judges verify the work is real.

### 8.1 Repository structure

```
pulso/
├── README.md
├── LICENSE
├── docs/
│   ├── architecture.md
│   ├── api.md
│   ├── design-guide.md
│   └── screenshots/
├── app/                  # Next.js / React frontend
├── convex/               # Convex backend
│   ├── schema.ts
│   ├── seed.ts
│   └── ...               # queries, mutations, actions
└── .env.example
```

### 8.2 README template

```markdown
# Pulso
Workload early-warning system for students and faculty.

## The problem
(2-3 sentences)

## Features
- Instructor-entered deadline management (create, edit, and delete)
- Student timeline and weekly load meter
- AI study assistant with structured, availability-aware study plans
- Faculty collision check with date suggestions
- Admin heatmap

## Demo
Live: <link> · Video: <link>
Test accounts: student@demo / instructor@demo / admin@demo

## Tech stack
- Convex (database, auth, server functions)
- Gemini API (primary LLM) and Mistral (fallback) for student study-plan generation
- Vercel (frontend hosting)

## How the load score works
(short formula explanation)

## Getting started
1. Clone the repo
2. Copy `.env.example` to `.env.local` and fill in values
3. Run `npm install`
4. Run `npx convex dev` to create or link your Convex project
5. Set the keys: `npx convex env set GEMINI_API_KEY <key>` and `npx convex env set MISTRAL_API_KEY <key>`
6. Seed demo data: `npx convex run seed:run`
7. Start the app with `npm run dev`

## Team
(names and roles)

## Rules compliance
(state that all code was written during the hackathon, plus any libraries and APIs used)
```

**Rationale:** the Fresh Code Rule is listed in the deck, so state clearly in the README that all code was written during the event, and list third-party libraries and APIs used.

### 8.3 Technical docs to write

- **architecture.md:** the diagram from section 4.1, plus one paragraph per component.
- **api.md:** each endpoint with an example request and response.
- **design-guide.md:** palette, typography, and component notes from section 5.

Example API doc entry:

```http
POST /api/collision-check
{ "section_id": 3, "date": "2026-12-03", "type": "exam", "weight_percent": 25 }

200 OK
{ "students_affected": 12, "total_students": 40,
  "suggestions": ["2026-12-10", "2026-12-08", "2026-12-11"] }
```

### 8.4 Git and commit conventions

- Branches: `main` (always deployable), plus `feat/<name>` and `fix/<name>`.
- Commit messages: `feat: add collision check endpoint`, `fix: handle null due_date`.
- Commit early and often with every member contributing, since the commit history shows team participation.
- Never commit secrets. Use `.env` and keep `.env.example` in the repo.

### 8.5 User guide (one page per role)

- **Student:** read the load meter, browse confirmed deadlines on the timeline, provide study availability, generate and adjust an AI-assisted study plan.
- **Instructor:** manually add, edit, or delete section deadlines, read the collision banner, pick a suggested date or keep the intended date.
- **Admin:** read the heatmap, filter by program and year level.

### 8.6 Pitch deck outline (PowerPoint)

1. Title and hook
2. The problem (week 8 story)
3. Insight: warn the people who set deadlines
4. Solution overview
5. Live demo or screenshots (student, instructor, admin)
6. How it works (load score and architecture)
7. Privacy and ethics
8. Impact and who benefits
9. What's next (calendar sync, LMS integration)
10. Team and links (repo, live demo)

Keep slides visual and light on text. Reuse the palette and typography from section 5.

### 8.7 Pre-submission checklist

- [ ] Repo is public and the README is complete
- [ ] `.env.example` present and no secrets committed
- [ ] App deployed and the live link works
- [ ] Test accounts documented
- [ ] Screenshots added to `docs/screenshots/`
- [ ] Deck exported as .pptx
- [ ] Both submitted in Discord before **Nov 12, 12 NN**
- [ ] Demo rehearsed with a fallback ready

---

## 9. Next Steps

1. Confirm the Tech Stack and Platform Usage rules with the organizers.
2. Lock the stack and split roles.
3. Write the schema and seed script first, since everything else depends on the data.
