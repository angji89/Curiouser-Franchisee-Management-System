# Bright Beginnings — Franchisee Management System

## Technical Specification (v1 — Prototype Scope)

This document covers the database schemas, API endpoints, and implementation notes for the three core modules demonstrated in the prototype:

1. **Class Plans Library** (IP-protected curriculum distribution)
2. **Training Academy** (LMS with courses, quizzes, certificates)
3. **SOP Execution** (mobile-first checklists with photo proof)

Brand palette used throughout: `#f7f2eb` cream, `#c98b2c` ochre, `#5c707a` slate, `#bf5f49` coral, `#3f4d51` ink.

---

## 1. Shared Foundations

### Roles & Access

| Role | Description |
|---|---|
| `hq_admin` | Full access. Uploads class plans, publishes courses, defines SOP templates, reviews execution logs. |
| `franchise_owner` | Manages their own franchise, views all franchise data, assigns staff to courses. |
| `teacher` | End user. Views class plans (no download), takes courses, executes SOPs. |
| `auditor` | Read-only across all franchises for HQ compliance review. |

### Core User Table

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT UNIQUE NOT NULL,
  name TEXT NOT NULL,
  role TEXT NOT NULL CHECK (role IN ('hq_admin','franchise_owner','teacher','auditor')),
  franchise_id UUID REFERENCES franchises(id),
  avatar_url TEXT,
  active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT now(),
  last_login_at TIMESTAMPTZ
);

CREATE TABLE franchises (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  code TEXT UNIQUE NOT NULL,        -- e.g. #047
  name TEXT NOT NULL,                -- Selangor Central
  region TEXT,
  owner_user_id UUID REFERENCES users(id),
  contract_start DATE,
  status TEXT DEFAULT 'active'
);
```

---

## 2. Module 1 — Class Plans Library

### Overview

Hierarchy: **Theme → Month → Week → Lesson Plan**. HQ uploads plans, franchisees view them in an IP-protected viewer.

### Database Schema

```sql
CREATE TABLE class_themes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  slug TEXT UNIQUE NOT NULL,         -- 'drama', 'music', 'stem'
  name TEXT NOT NULL,                 -- 'Drama & Theatre'
  description TEXT,
  icon TEXT,                          -- emoji or icon key
  brand_color TEXT,
  display_order INT DEFAULT 0,
  active BOOLEAN DEFAULT true
);

CREATE TABLE class_plan_folders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  theme_id UUID NOT NULL REFERENCES class_themes(id) ON DELETE CASCADE,
  year INT NOT NULL,                  -- 2026
  month INT NOT NULL CHECK (month BETWEEN 1 AND 12),
  week_of_month INT NOT NULL CHECK (week_of_month BETWEEN 1 AND 5),
  UNIQUE (theme_id, year, month, week_of_month)
);

CREATE TABLE class_plans (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  folder_id UUID NOT NULL REFERENCES class_plan_folders(id) ON DELETE CASCADE,
  lesson_number INT NOT NULL,
  title TEXT NOT NULL,
  overview TEXT,
  age_group TEXT,                     -- '4-5'
  duration_minutes INT,
  storage_key TEXT NOT NULL,          -- S3/R2 object key for source PDF
  viewer_doc_id TEXT,                 -- reference to rendered/watermarked copy
  required_course_id UUID REFERENCES courses(id), -- must complete this course to view
  status TEXT DEFAULT 'published' CHECK (status IN ('draft','published','archived')),
  published_at TIMESTAMPTZ,
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_class_plans_folder ON class_plans(folder_id);
CREATE INDEX idx_folders_theme ON class_plan_folders(theme_id, year, month);

-- Audit every view
CREATE TABLE class_plan_views (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  plan_id UUID NOT NULL REFERENCES class_plans(id),
  user_id UUID NOT NULL REFERENCES users(id),
  ip_address INET,
  user_agent TEXT,
  session_duration_seconds INT,
  pages_viewed INT[],
  viewed_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_views_plan_time ON class_plan_views(plan_id, viewed_at DESC);

-- Log suspicious activity (attempted screenshots, dev tools, etc.)
CREATE TABLE class_plan_security_events (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  plan_id UUID REFERENCES class_plans(id),
  user_id UUID REFERENCES users(id),
  event_type TEXT NOT NULL,           -- 'right_click','ctrl_s','ctrl_p','print_screen','devtools_open','context_menu'
  metadata JSONB,
  occurred_at TIMESTAMPTZ DEFAULT now()
);
```

### API Endpoints

```http
# Browse hierarchy
GET /api/v1/themes
GET /api/v1/themes/:slug/folders?year=2026&month=4
GET /api/v1/folders/:id/plans

# Individual plan (returns metadata only, not content)
GET /api/v1/plans/:id
  → { id, title, overview, age_group, duration_minutes, required_course_completed, viewer_url }

# Viewer URL: short-lived signed URL to a dynamically watermarked, image-rendered version
GET /api/v1/plans/:id/viewer-session
  → {
      session_token: "eyJ...",        # expires in 15 min
      pages: [{ page: 1, image_url: "..." }, ...],  # rasterized PNG with embedded watermark
      watermark: "sarah.chen@... · 23 Apr 2026"
    }

# Log security events from the frontend
POST /api/v1/plans/:id/security-event
  body: { event_type, metadata }

# HQ admin — upload
POST /api/v1/plans            (hq_admin only)
PATCH /api/v1/plans/:id
DELETE /api/v1/plans/:id
```

### IP Protection Strategy — layered defense

No browser-based system can be 100% leak-proof (a phone camera will always work), but these layers raise the bar substantially and create a legal audit trail.

**Frontend layer (demonstrated in prototype):**
- `user-select: none` on all protected content
- Block right-click, copy, drag, Ctrl+S / Ctrl+P / Ctrl+C / Ctrl+A / F12 / PrintScreen
- Detect DevTools (devtools-detect library) and blank the content
- Dynamically rendered watermark overlay with user email + timestamp, repeated across the page at ~30° rotation with low opacity
- Content served as rasterized images, not selectable text
- CSS `@media print` hides all protected content

**Backend layer:**
- Source PDFs never sent to client — server renders each page as a watermarked PNG/WebP on demand
- Short-lived signed URLs (15 min) tied to user session
- Rate-limit page-fetch calls to prevent automated scraping
- Log every view to `class_plan_views`; log anomalies to `class_plan_security_events`
- Admin dashboard surfaces users with unusual view patterns (e.g., > 50 views/day, rapid sequential page fetches)

**Legal & organizational layer:**
- Franchise agreement explicitly prohibits reproduction/distribution
- Unique per-user watermark makes any leaked screenshot traceable
- Terms of use modal on first view

### Suggested File Storage Layout (S3/R2)

```
/curriculum/
  /drama/
    /2026/
      /04/
        /week-2/
          lesson-1_ocean-pollution/
            source.pdf               (private, never served)
            rendered/
              page-1.png
              page-2.png
              ...
          lesson-2_turtles-rescue/
            ...
```

---

## 3. Module 2 — Training Academy

### Overview

Coursera-style LMS: **Course → Module → Lesson** (video, slides, or quiz). Progress tracked per user per lesson. Certificates generated on course completion. Required courses gate access to curriculum.

### Database Schema

```sql
CREATE TABLE courses (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title TEXT NOT NULL,
  subtitle TEXT,
  description TEXT,
  cover_image_url TEXT,
  level TEXT CHECK (level IN ('Beginner','Intermediate','Advanced')),
  total_hours NUMERIC(4,2),
  instructor_name TEXT,
  is_required BOOLEAN DEFAULT false,
  required_role TEXT[],              -- ['teacher','franchise_owner']
  pass_threshold NUMERIC(3,2) DEFAULT 0.70,
  status TEXT DEFAULT 'draft' CHECK (status IN ('draft','published','archived')),
  published_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE modules (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  course_id UUID NOT NULL REFERENCES courses(id) ON DELETE CASCADE,
  title TEXT NOT NULL,
  display_order INT NOT NULL,
  UNIQUE (course_id, display_order)
);

CREATE TABLE lessons (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  module_id UUID NOT NULL REFERENCES modules(id) ON DELETE CASCADE,
  title TEXT NOT NULL,
  description TEXT,
  display_order INT NOT NULL,
  lesson_type TEXT NOT NULL CHECK (lesson_type IN ('video','slides','quiz','reading')),
  duration_minutes INT,
  video_url TEXT,                     -- for video lessons (Mux, Cloudflare Stream, etc.)
  slides_url TEXT,                    -- PPT/PDF for slide lessons
  content_body TEXT,                  -- markdown for reading-only lessons
  UNIQUE (module_id, display_order)
);

CREATE TABLE lesson_resources (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  lesson_id UUID NOT NULL REFERENCES lessons(id) ON DELETE CASCADE,
  title TEXT,
  file_url TEXT,
  file_type TEXT,                     -- 'pdf','pptx','image','link'
  file_size_bytes BIGINT
);

-- Quizzes
CREATE TABLE quizzes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  lesson_id UUID NOT NULL REFERENCES lessons(id) ON DELETE CASCADE UNIQUE,
  title TEXT,
  pass_threshold NUMERIC(3,2) DEFAULT 0.70,
  max_attempts INT DEFAULT 3,
  show_explanations BOOLEAN DEFAULT true
);

CREATE TABLE quiz_questions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  quiz_id UUID NOT NULL REFERENCES quizzes(id) ON DELETE CASCADE,
  question TEXT NOT NULL,
  display_order INT,
  question_type TEXT DEFAULT 'single_choice',
  options JSONB NOT NULL,             -- [{ "id": "a", "text": "2–5 min" }, ...]
  correct_option_ids TEXT[] NOT NULL,
  explanation TEXT,
  points INT DEFAULT 1
);

-- Progress tracking
CREATE TABLE enrollments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id),
  course_id UUID NOT NULL REFERENCES courses(id),
  enrolled_at TIMESTAMPTZ DEFAULT now(),
  last_accessed_at TIMESTAMPTZ,
  progress_percent INT DEFAULT 0,
  status TEXT DEFAULT 'in_progress' CHECK (status IN ('in_progress','completed','expired')),
  completed_at TIMESTAMPTZ,
  UNIQUE (user_id, course_id)
);

CREATE TABLE lesson_progress (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  enrollment_id UUID NOT NULL REFERENCES enrollments(id) ON DELETE CASCADE,
  lesson_id UUID NOT NULL REFERENCES lessons(id),
  status TEXT DEFAULT 'not_started' CHECK (status IN ('not_started','in_progress','completed')),
  progress_percent INT DEFAULT 0,     -- 0–100 for videos: % watched
  time_spent_seconds INT DEFAULT 0,
  last_position_seconds INT DEFAULT 0, -- resume position for video
  started_at TIMESTAMPTZ,
  completed_at TIMESTAMPTZ,
  UNIQUE (enrollment_id, lesson_id)
);

CREATE INDEX idx_lesson_progress_enrollment ON lesson_progress(enrollment_id);

CREATE TABLE quiz_attempts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id),
  quiz_id UUID NOT NULL REFERENCES quizzes(id),
  attempt_number INT NOT NULL,
  started_at TIMESTAMPTZ DEFAULT now(),
  submitted_at TIMESTAMPTZ,
  score NUMERIC(5,2),                 -- 0.00–1.00
  passed BOOLEAN,
  answers JSONB                        -- [{ question_id, selected_option_ids }, ...]
);

-- Certificates
CREATE TABLE certificates (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  serial TEXT UNIQUE NOT NULL,        -- 'BB-2026-4872'
  user_id UUID NOT NULL REFERENCES users(id),
  course_id UUID NOT NULL REFERENCES courses(id),
  issued_at TIMESTAMPTZ DEFAULT now(),
  pdf_url TEXT,                        -- generated on completion
  verification_url TEXT                -- public verify page
);

CREATE INDEX idx_certs_user ON certificates(user_id);
```

### API Endpoints

```http
# Discovery
GET  /api/v1/courses?filter=required|in_progress|completed
GET  /api/v1/courses/:id                  # full structure w/ modules & lessons
GET  /api/v1/courses/:id/progress          # current user's progress
POST /api/v1/courses/:id/enroll

# Lessons
GET  /api/v1/lessons/:id
POST /api/v1/lessons/:id/progress          # body: { progress_percent, time_spent_delta, last_position }
POST /api/v1/lessons/:id/complete

# Quizzes
GET  /api/v1/quizzes/:id
POST /api/v1/quizzes/:id/attempts          # body: { answers: [...] }
   → { score, passed, per_question_results: [...] }

# Certificates
GET  /api/v1/me/certificates
GET  /api/v1/certificates/:serial          # public verify (no auth)
GET  /api/v1/certificates/:id/download     # PDF stream

# Gating check — used by Class Plans module
GET  /api/v1/me/required-courses-status
  → { completed_ids: [...], pending_required_ids: [...] }
```

### Certificate PDF Generation

Use a headless renderer (Puppeteer, or server-side with a library like `@react-pdf/renderer` or Chromium-via-Lambda). Template includes:
- Franchisee name
- Course title
- Issue date
- Unique serial (e.g., `BB-2026-4872`)
- QR code linking to `/verify/:serial` for authenticity

### Gating Logic (Access Control)

When a user tries to view a class plan:

```
1. Fetch plan → if plan.required_course_id IS NULL, allow view
2. Fetch user's enrollment for that course
3. If enrollment.status = 'completed' → allow
4. Otherwise → redirect to course page with message
   "Complete '{course title}' to unlock this curriculum."
```

The prototype displays a hint on locked class plans. Extend this to show a disabled card with a "Complete training first" CTA.

---

## 4. Module 3 — SOP Execution

### Overview

HQ defines **SOP templates** (ordered steps with optional photo proof requirements). Teachers **execute** them for a given class session, checking off steps and uploading photos. Everything is logged for HQ review.

### Database Schema

```sql
CREATE TABLE sop_templates (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title TEXT NOT NULL,
  description TEXT,
  linked_class_plan_id UUID REFERENCES class_plans(id),  -- optional link
  linked_theme_id UUID REFERENCES class_themes(id),       -- optional broader link
  duration_minutes INT,
  status TEXT DEFAULT 'draft' CHECK (status IN ('draft','published','archived')),
  version INT DEFAULT 1,
  published_at TIMESTAMPTZ,
  created_by UUID REFERENCES users(id),
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE sop_materials (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  template_id UUID NOT NULL REFERENCES sop_templates(id) ON DELETE CASCADE,
  label TEXT NOT NULL,
  quantity TEXT,
  display_order INT
);

CREATE TABLE sop_steps (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  template_id UUID NOT NULL REFERENCES sop_templates(id) ON DELETE CASCADE,
  display_order INT NOT NULL,
  title TEXT NOT NULL,
  description TEXT,
  estimated_time TEXT,                -- 'before','5 min','10 min'
  requires_photo BOOLEAN DEFAULT false,
  photo_prompt TEXT,                   -- 'Photo of prepared ocean setup'
  reference_image_url TEXT,            -- example image for teachers
  reference_video_url TEXT,
  UNIQUE (template_id, display_order)
);

-- Executions — one per class session
CREATE TABLE sop_executions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  template_id UUID NOT NULL REFERENCES sop_templates(id),
  template_version INT NOT NULL,       -- pin version at execution time
  teacher_id UUID NOT NULL REFERENCES users(id),
  franchise_id UUID NOT NULL REFERENCES franchises(id),
  class_plan_id UUID REFERENCES class_plans(id),
  session_date DATE NOT NULL,
  started_at TIMESTAMPTZ,
  completed_at TIMESTAMPTZ,
  status TEXT DEFAULT 'in_progress' CHECK (status IN ('in_progress','completed','abandoned')),
  steps_completed INT DEFAULT 0,
  total_steps INT NOT NULL,
  completion_percent INT DEFAULT 0,
  overall_notes TEXT
);

CREATE INDEX idx_executions_franchise_date ON sop_executions(franchise_id, session_date DESC);
CREATE INDEX idx_executions_teacher ON sop_executions(teacher_id);

CREATE TABLE sop_execution_steps (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  execution_id UUID NOT NULL REFERENCES sop_executions(id) ON DELETE CASCADE,
  step_id UUID NOT NULL REFERENCES sop_steps(id),
  completed BOOLEAN DEFAULT false,
  completed_at TIMESTAMPTZ,
  notes TEXT,
  UNIQUE (execution_id, step_id)
);

CREATE TABLE sop_execution_photos (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  execution_step_id UUID NOT NULL REFERENCES sop_execution_steps(id) ON DELETE CASCADE,
  storage_key TEXT NOT NULL,
  thumbnail_url TEXT,
  full_url TEXT,
  file_size_bytes BIGINT,
  captured_at TIMESTAMPTZ DEFAULT now(),
  gps_lat NUMERIC(10,7),
  gps_lng NUMERIC(10,7)
);
```

### API Endpoints

```http
# Browse templates
GET  /api/v1/sop-templates?class_plan_id=&theme_id=
GET  /api/v1/sop-templates/:id

# Executions
POST /api/v1/sop-executions
  body: { template_id, class_plan_id?, session_date }
  → creates execution + pre-creates all sop_execution_steps with completed=false

GET  /api/v1/sop-executions/:id

# Step-level updates (sent as teacher progresses)
PATCH /api/v1/sop-executions/:id/steps/:step_id
  body: { completed: true, notes: "..." }

# Photo upload (two-step: presign + upload)
POST /api/v1/sop-executions/:id/steps/:step_id/photos/presign
  → { upload_url, storage_key }
# Client then PUTs the photo directly to storage_key
POST /api/v1/sop-executions/:id/steps/:step_id/photos
  body: { storage_key }

# Submit
POST /api/v1/sop-executions/:id/submit
  → validates: all required steps done, all photo checkpoints have ≥1 photo
  → status = 'completed'

# HQ review
GET  /api/v1/admin/sop-executions?franchise_id=&from=&to=
GET  /api/v1/admin/compliance-report?franchise_id=
```

### Validation Logic on Submit

```python
def validate_execution_submission(execution_id):
    execution = get_execution(execution_id)
    steps     = get_execution_steps(execution_id)

    # 1) every step checked off
    if not all(s.completed for s in steps):
        raise ValidationError("Some steps are not marked complete")

    # 2) every photo-required step has at least one photo
    for s in steps:
        template_step = get_template_step(s.step_id)
        if template_step.requires_photo:
            photos = get_photos(s.id)
            if not photos:
                raise ValidationError(
                  f"Step '{template_step.title}' requires a photo"
                )

    return True
```

---

## 5. Cross-Module: Required Course → Curriculum Unlock

This is the **access control** flow you asked for: teachers must complete required courses before unlocking features (specifically, viewing class plans for that theme).

### Flow

```
┌────────────────────┐       ┌──────────────────────┐       ┌─────────────────┐
│ class_plans        │       │ courses              │       │ enrollments     │
│                    │       │                      │       │                 │
│ required_course_id ├──────►│ id                   │◄──────┤ course_id       │
│                    │       │ is_required          │       │ user_id         │
│                    │       │                      │       │ status          │
└────────────────────┘       └──────────────────────┘       └─────────────────┘
```

### Middleware

```typescript
async function canViewClassPlan(userId: string, planId: string): Promise<boolean> {
  const plan = await db.classPlans.findById(planId);
  if (!plan.requiredCourseId) return true;

  const enrollment = await db.enrollments.findOne({
    userId,
    courseId: plan.requiredCourseId,
    status: 'completed'
  });

  return !!enrollment;
}
```

### UX for Locked Plans

In the prototype, extend `ClassPlansWeekView` to check the user's completion status. Locked plans render with a lock icon and, on click, open a modal: *"To access Drama curriculum, complete the Drama Class Delivery Masterclass"* with a direct link to the course.

---

## 6. Recommended Tech Stack

| Layer | Recommendation |
|---|---|
| **Frontend** | Next.js (App Router) + React + Tailwind (matching prototype) |
| **Mobile** | Progressive Web App first (prototype is already mobile-responsive); React Native if a native app is needed later |
| **Backend API** | Node.js / Fastify or NestJS · alternatively Python / FastAPI |
| **Database** | PostgreSQL (schemas above written for PG) |
| **File Storage** | Cloudflare R2 or AWS S3 with signed URLs |
| **Video hosting** | Mux or Cloudflare Stream (both support DRM and thumbnailing) |
| **Auth** | Auth0, Clerk, or Supabase Auth |
| **PDF rendering** | Puppeteer via serverless function, or `@react-pdf/renderer` |
| **Watermarking** | Server-side with Sharp (Node) or Pillow (Python) to burn watermark into rendered PNGs |

---

## 7. Phase-1 Milestones (suggested 10-week build)

| Week | Deliverable |
|---|---|
| 1–2 | Auth, user/franchise tables, admin console shell |
| 3–4 | Class Plans library upload + hierarchical browse |
| 5 | IP-protected viewer (raster rendering + dynamic watermark + anti-copy) |
| 6–7 | Training Academy: courses, modules, lessons, progress, video player |
| 8 | Quizzes + certificate generation |
| 9 | SOP templates + execution + photo upload |
| 10 | HQ review dashboards, compliance reports, polish |

---

## 8. Open Questions for You

A few things worth deciding before build:

1. **Languages** — is this single-language (English), or do you need multilingual curriculum (e.g. Bahasa Malaysia, Mandarin)?
2. **Offline access** — should teachers be able to pre-download SOPs (no class plans, since those are IP-protected) for classrooms with poor WiFi?
3. **Parent-facing module** — out of scope for now, but worth flagging if you want photos/progress to flow to parents eventually.
4. **Compliance region** — Malaysia PDPA applies; if expanding to EU, GDPR rules around children's data are stricter.
5. **Video DRM level** — basic signed URLs vs full DRM (Widevine/FairPlay). Full DRM adds cost; signed URLs usually sufficient for internal training.

---

*This spec covers the prototype demonstrated. Full production build would add: notifications, admin dashboards, bulk import tools, analytics, and audit logs — all straightforward extensions of the schemas above.*
