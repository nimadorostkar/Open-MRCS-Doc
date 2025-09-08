### Overview

Below is a complete backend technical plan for powering the OpenMRCS dashboard and study platform seen at [OpenMRCS Dashboard](https://open-mrcs-frontend-client.vercel.app/dashboard/). It covers Django app structure, core data models, API endpoints, authentication/authorization, analytics, recommendations, admin/CMS, and operational concerns. It is designed to fully support features shown on the dashboard: user profile (name/email), Practice Mode, Exam Mode, Question Bank progress, performance metrics, recent activity, suggested topics, and high‑yield resources.

### High-level architecture

- **Framework**: Django + Django REST Framework (DRF)
- **Database**: PostgreSQL
- **Cache/Queues**: Redis (Django cache + Celery broker)
- **Async tasks**: Celery + Celery Beat (scheduled jobs)
- **Storage**: S3-compatible for media (videos/thumbnails) or external links
- **Auth**: JWT (SimpleJWT) + optional social logins
- **API**: Versioned REST, `api/v1/...`
- **Admin/CMS**: Django Admin or Wagtail (for Resources/CMS pages)
- **Observability**: Django signals + OpenTelemetry + Sentry
- **Security**: DRF throttling, IP rate limits, object-level permissions

### Django apps

- `accounts`: users, profiles, roles, settings, GDPR controls
- `taxonomy`: categories, subcategories, tags
- `questions`: question bank, answers, explanations, media, difficulty
- `practice`: practice sessions, attempts, per-question feedback
- `exams`: mock exams (paper 1/2), templates, timed sessions, submissions
- `analytics`: scores, progress, aggregates, study time, per-category metrics
- `recommendations`: suggested topics, rules/ML, content targeting
- `resources`: videos, seminars, articles, playlists, metadata
- `notifications`: in-app, email, optional push
- `billing` (optional): plans, subscriptions, entitlements
- `cms`: static pages, featured carousels, legal documents
- `audit`: event log, staff content changes, data exports

---

## Data model (core)

```python
from django.contrib.auth.models import AbstractUser
from django.db import models

# accounts
class User(AbstractUser):
    email = models.EmailField(unique=True)
    # username optional; prefer email as primary
    is_email_verified = models.BooleanField(default=False)
    # roles: user, editor, examiner, admin
    role = models.CharField(max_length=20, default="user")

class Profile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE, related_name="profile")
    display_name = models.CharField(max_length=120, blank=True)
    avatar = models.URLField(blank=True)
    timezone = models.CharField(max_length=64, default="UTC")
    # preferences
    prefers_explanations = models.BooleanField(default=True)

# taxonomy
class Category(models.Model):
    name = models.CharField(max_length=120, unique=True)
    slug = models.SlugField(unique=True)
    description = models.TextField(blank=True)
    order = models.PositiveIntegerField(default=0)
    # e.g., "Anatomy & Embryology", "Orthopaedic Surgery"

class Subcategory(models.Model):
    category = models.ForeignKey(Category, on_delete=models.CASCADE, related_name="subcategories")
    name = models.CharField(max_length=120)
    slug = models.SlugField()
    description = models.TextField(blank=True)
    order = models.PositiveIntegerField(default=0)

# questions
class Question(models.Model):
    SINGLE = "single"
    MULTI = "multi"
    TYPE_CHOICES = [(SINGLE, "Single best answer"), (MULTI, "Multiple response")]
    stem = models.TextField()
    type = models.CharField(max_length=10, choices=TYPE_CHOICES, default=SINGLE)
    category = models.ForeignKey(Category, on_delete=models.PROTECT, related_name="questions")
    subcategory = models.ForeignKey(Subcategory, on_delete=models.PROTECT, null=True, blank=True)
    difficulty = models.PositiveSmallIntegerField(default=3)  # 1..5
    explanation = models.TextField(blank=True)
    reference = models.TextField(blank=True)  # citations/guidelines
    is_active = models.BooleanField(default=True)
    created_by = models.ForeignKey(User, on_delete=models.SET_NULL, null=True, related_name="+")
    updated_at = models.DateTimeField(auto_now=True)

class AnswerOption(models.Model):
    question = models.ForeignKey(Question, on_delete=models.CASCADE, related_name="options")
    text = models.TextField()
    is_correct = models.BooleanField(default=False)
    order = models.PositiveSmallIntegerField(default=0)

class QuestionMedia(models.Model):
    question = models.ForeignKey(Question, on_delete=models.CASCADE, related_name="media")
    url = models.URLField()
    kind = models.CharField(max_length=20, default="image")  # image, audio, video

# practice
class PracticeSession(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name="practice_sessions")
    categories = models.ManyToManyField(Category, blank=True)
    subcategories = models.ManyToManyField(Subcategory, blank=True)
    question_count = models.PositiveIntegerField()
    include_explanations = models.BooleanField(default=True)
    started_at = models.DateTimeField(auto_now_add=True)
    ended_at = models.DateTimeField(null=True, blank=True)

class PracticeAttempt(models.Model):
    session = models.ForeignKey(PracticeSession, on_delete=models.CASCADE, related_name="attempts")
    question = models.ForeignKey(Question, on_delete=models.PROTECT)
    selected_option_ids = models.JSONField(default=list)
    is_correct = models.BooleanField()
    time_spent_sec = models.PositiveIntegerField(default=0)
    answered_at = models.DateTimeField(auto_now_add=True)

# exams
class ExamTemplate(models.Model):
    name = models.CharField(max_length=120)  # "Exam A, Paper 2"
    paper = models.CharField(max_length=32)  # "Paper 1" | "Paper 2"
    duration_minutes = models.PositiveIntegerField(default=180)
    total_questions = models.PositiveIntegerField()
    description = models.TextField(blank=True)
    is_active = models.BooleanField(default=True)

class ExamTemplateItem(models.Model):
    template = models.ForeignKey(ExamTemplate, on_delete=models.CASCADE, related_name="items")
    question = models.ForeignKey(Question, on_delete=models.PROTECT)
    order = models.PositiveIntegerField()

class ExamSession(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name="exam_sessions")
    template = models.ForeignKey(ExamTemplate, on_delete=models.PROTECT)
    started_at = models.DateTimeField(auto_now_add=True)
    submitted_at = models.DateTimeField(null=True, blank=True)
    score_percent = models.DecimalField(max_digits=5, decimal_places=2, null=True, blank=True)
    time_spent_sec = models.PositiveIntegerField(default=0)

class ExamAttempt(models.Model):
    session = models.ForeignKey(ExamSession, on_delete=models.CASCADE, related_name="attempts")
    question = models.ForeignKey(Question, on_delete=models.PROTECT)
    selected_option_ids = models.JSONField(default=list)
    is_correct = models.BooleanField()
    time_spent_sec = models.PositiveIntegerField(default=0)

# analytics
class DailyStudyStat(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name="daily_stats")
    date = models.DateField()
    time_spent_sec = models.PositiveIntegerField(default=0)
    questions_answered = models.PositiveIntegerField(default=0)
    correct_answers = models.PositiveIntegerField(default=0)

class CategoryAggregate(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name="category_aggregates")
    category = models.ForeignKey(Category, on_delete=models.CASCADE)
    answered = models.PositiveIntegerField(default=0)
    correct = models.PositiveIntegerField(default=0)
    last_practiced_at = models.DateTimeField(null=True, blank=True)

# recommendations
class SuggestedTopic(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name="suggestions")
    category = models.ForeignKey(Category, on_delete=models.CASCADE)
    reason = models.CharField(max_length=200)  # "Low score"
    score_percent = models.DecimalField(max_digits=5, decimal_places=2, null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

# resources
class Resource(models.Model):
    VIDEO = "video"
    SEMINAR = "seminar"
    ARTICLE = "article"
    kind = models.CharField(max_length=20, choices=[(VIDEO,"Video"),(SEMINAR,"Seminar"),(ARTICLE,"Article")])
    title = models.CharField(max_length=200)
    author = models.CharField(max_length=120, blank=True)
    duration_sec = models.PositiveIntegerField(default=0)
    category = models.ForeignKey(Category, null=True, blank=True, on_delete=models.SET_NULL)
    url = models.URLField()          # external streaming or internal
    thumbnail_url = models.URLField(blank=True)
    is_featured = models.BooleanField(default=False)
    published_at = models.DateTimeField(null=True, blank=True)

# notifications
class Notification(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name="notifications")
    title = models.CharField(max_length=200)
    body = models.TextField()
    is_read = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)

# billing (optional)
class Plan(models.Model):
    code = models.CharField(max_length=50, unique=True)
    name = models.CharField(max_length=120)
    price_cents = models.PositiveIntegerField()
    interval = models.CharField(max_length=10, default="monthly")
    features = models.JSONField(default=dict)

class Subscription(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE, related_name="subscriptions")
    plan = models.ForeignKey(Plan, on_delete=models.PROTECT)
    provider = models.CharField(max_length=20, default="stripe")
    provider_sub_id = models.CharField(max_length=200)
    status = models.CharField(max_length=20, default="active")
    current_period_end = models.DateTimeField()
```

Notes:
- Use DB constraints and indexes on foreign keys and common filters (`user`, `category`, `created_at`).
- For multi-correct questions, `selected_option_ids` supports arrays.

---

## API design (v1)

Base: `/api/v1/...` (JWT Bearer auth except for public endpoints)

### Auth and profile

- POST `/auth/register` — email, password; verify email
- POST `/auth/login` — returns `access`, `refresh`
- POST `/auth/refresh` — rotate tokens
- POST `/auth/logout`
- POST `/auth/forgot-password`, POST `/auth/reset-password`
- GET `/me` — user + profile
- PATCH `/me` — update profile prefs (e.g., display name)
- GET `/me/notifications` | POST mark read

Example:
```json
// POST /auth/login
{ "email": "negin.jahan@email.com", "password": "secret" }

// 200
{ "access": "jwt...", "refresh": "jwt..." }
```

### Dashboard aggregates (single round-trip for the dashboard UI)

- GET `/dashboard` — returns:
  - overall progress: completed categories count, total categories
  - performance: average score %, total correct/incorrect
  - study time: hours this week, delta vs last week
  - recent activity: last practice session summary, last exam summary
  - suggested topics: 3-5 low-score categories with CTA
  - featured resources

Example:
```json
{
  "overall_progress": { "completed": 18, "total": 25 },
  "performance": { "avg_score_percent": 78, "correct": 245, "incorrect": 69 },
  "study_time": { "hours_this_week": 14.5, "delta_vs_last_week": 2.5 },
  "recent_activity": {
    "last_practice": { "days_ago": 2, "score_percent": 80, "correct": 20, "incorrect": 5, "categories": ["Anatomy & Embryology","Cancer and Palliative Care"] },
    "last_exam": { "days_ago": 5, "exam": "Exam A, Paper 2", "score_percent": 72, "time_spent_min": 165, "answered": 180 }
  },
  "suggested_topics": [{ "category": "Anatomy & Embryology", "score_percent": 54, "reason": "Low Score" }],
  "featured_resources": [{ "kind": "video", "title": "Surgical Anatomy of the Abdomen", "duration_sec": 750, "url": "..." }]
}
```

### Taxonomy and question bank

- GET `/categories` — list with progress per user
- GET `/categories/{slug}` — detail + subcategories
- GET `/questions` — filters: category, subcategory, difficulty, active
- GET `/questions/{id}` — includes options, explanation (if entitled)
- Staff:
  - POST `/questions` (create), PUT/PATCH `/questions/{id}`, DELETE `/questions/{id}`
  - Bulk import `/questions/import` (CSV/JSON)

### Practice Mode

- POST `/practice/sessions` — create with payload:
  - `categories: [slug]`, `subcategories: [slug]`, `question_count`, `include_explanations`
  - backend selects randomized set (deterministic seed saved)
- GET `/practice/sessions/{id}` — session detail + question list
- POST `/practice/sessions/{id}/attempts` — submit per-question
  - request: `question_id`, `selected_option_ids`, `time_spent_sec`
  - response: correctness, explanation (if enabled), correct options
- POST `/practice/sessions/{id}/complete` — finalize session
- GET `/practice/sessions` — list recent with scores
- GET `/practice/summary` — aggregate correctness per category

Example:
```json
// POST /practice/sessions/{id}/attempts
{ "question_id": 123, "selected_option_ids": [456], "time_spent_sec": 42 }

// 200
{ "is_correct": true, "correct_option_ids": [456], "explanation": "..." }
```

### Exam Mode (mock exams)

- GET `/exams/templates` — available exams (Paper 1/2, duration, question count)
- POST `/exams/sessions` — start session: `template_id`
- GET `/exams/sessions/{id}` — includes ordered questions
- POST `/exams/sessions/{id}/attempts` — per-question or batched
- POST `/exams/sessions/{id}/submit` — submit entire exam
  - backend calculates `score_percent`, persists timing, locks session
- GET `/exams/sessions` — history with scores and durations
- GET `/exams/sessions/{id}/review` — show questions, user answers, correct answers, explanations (post-submission)

### Analytics and progress

- GET `/analytics/overview` — average score, correct/incorrect totals
- GET `/analytics/study-time?range=week` — time series for charts
- GET `/analytics/categories` — per-category answered/correct, last practiced
- GET `/analytics/heatmap` — day-level study counts
- GET `/analytics/trends` — performance over time

### Recommendations

- GET `/recommendations/topics` — suggested categories with reasons
- POST `/recommendations/dismiss` — dismiss suggestion
- Scheduled job recomputes suggestions daily:
  - rule: categories with score < threshold and enough attempts
  - or unsampled categories with zero attempts

### Resources

- GET `/resources?featured=true` — featured
- GET `/resources?category=anatomy-embryology` — filter
- GET `/resources/{id}`
- Staff CRUD for content and feature flags
- Optional signed URLs if private assets

### Notifications

- GET `/notifications`
- POST `/notifications/{id}/read`
- Staff: create broadcast or targeted notifications

### Billing (optional)

- GET `/billing/plans`
- POST `/billing/checkout` — initialize provider session
- Webhooks `/billing/webhooks/provider` — update `Subscription`
- GET `/billing/subscription` — user entitlement
- Middleware/permission gate to restrict premium features if used

---

## Key service logic

- Practice attempt:
  - compute `is_correct` by set-equality of `selected_option_ids` vs correct options
  - update `DailyStudyStat`, `CategoryAggregate`
- Exam submission:
  - lock session, compute score, record `time_spent_sec` as sum of attempts
- Progress:
  - “category completed” rule configurable (e.g., answered >= N with score >= X)
- Study time:
  - accumulate from attempts’ `time_spent_sec`; also allow heartbeat pings
- Suggestions:
  - pick 3-5 categories with lowest score and sufficient attempts; include “Not started” categories if needed

---

## Permissions and roles

- Roles: `user`, `editor` (content), `examiner` (exam templates), `admin`
- DRF permissions:
  - Users read their own sessions/analytics
  - Editors can CRUD `Question`, `Resource`
  - Examiners manage `ExamTemplate`
  - Admin full access
- Object-level checks on user-owned data

---

## Performance and scale

- Caching:
  - `/dashboard` response cached per-user (short TTL, e.g., 30–60s)
  - Category aggregates denormalized in `CategoryAggregate`
- Query optimization:
  - `select_related` on FK chains; `prefetch_related` for options/media
  - partial indexes on `is_active`, `category`
- Rate limiting:
  - Auth endpoints and attempts endpoints
- Bulk operations:
  - Bulk import for questions (CSV/JSON), Celery task with validation report

---

## Admin & CMS

- Django Admin for all models with list filters, search by stem/title
- Rich text for `explanation` and `resources`
- Content workflows:
  - draft/published flags, `updated_at`, `created_by`
  - audit trails in `audit` app
- Legal pages stored in `cms` with slug routes

---

## Observability & quality

- Sentry for errors
- OpenTelemetry tracing for slow endpoints (`/dashboard`, `/exams/submit`)
- Logging: request IDs, user ID, session IDs
- Tests:
  - Unit tests for scoring and aggregates
  - API tests for core flows (practice, exam)
  - Property tests for answer correctness logic
- Seed scripts:
  - Categories (25), questions, sample exams, resources

---

## API response contracts (additional examples)

```json
// GET /categories
[
  { "slug": "anatomy-embryology", "name": "Anatomy & Embryology", "progress": { "answered": 320, "correct": 172, "score_percent": 53.75 } },
  { "slug": "orthopaedic-surgery", "name": "Orthopaedic Surgery", "progress": { "answered": 110, "correct": 69, "score_percent": 62.73 } }
]

// GET /exams/templates
[
  { "id": 1, "name": "Exam A, Paper 2", "paper": "Paper 2", "duration_minutes": 180, "total_questions": 180, "description": "" }
]

// GET /practice/sessions/{id}
{
  "id": 42,
  "question_ids": [101,102,103],
  "config": { "include_explanations": true, "categories": ["anatomy-embryology"] }
}

// GET /exams/sessions/{id}/review
{
  "id": 77,
  "template": { "name": "Exam A, Paper 2" },
  "score_percent": 72,
  "attempts": [
    {
      "question_id": 1001,
      "selected_option_ids": [9001],
      "correct_option_ids": [9001],
      "is_correct": true,
      "explanation": "..."
    }
  ]
}
```

---

## Security and compliance

- JWT with short-lived access, refresh rotation, device revocation list
- Email verification; password rules; IP throttling
- PII minimization; GDPR endpoints:
  - GET `/me/export`, DELETE `/me` (data deletion)
- Audit logs for content changes and exam/session mutations

---

## DevOps

- Environments: dev/staging/prod
- Migrations and seed scripts
- CI: tests + lint + type checks (mypy)
- CD: blue/green or rolling; run `migrate`, collectstatic, warm caches
- Backups: daily PG dumps, S3 lifecycle policies
- Feature flags for new modules (e.g., billing)

---

## Frontend integration notes for the dashboard UI

To power the widgets seen at [OpenMRCS Dashboard](https://open-mrcs-frontend-client.vercel.app/dashboard/):

- Fetch `/api/v1/dashboard` on load; drives:
  - Overall Progress: `completed` of `total`
  - Performance: `avg_score_percent`, `correct`, `incorrect`
  - Study Time: `hours_this_week`, `delta_vs_last_week`
  - Recent Activity: `last_practice`, `last_exam`
  - Suggested Topics: array for CTAs (“Practice Now”)
  - High Yield Resources: `featured_resources`
- Practice Mode flow:
  - create session → stream questions → submit attempts → complete → refresh `/dashboard`
- Exam Mode flow:
  - start session (timed) → submit → review → refresh `/dashboard`

---

## Implementation checklist (phased)

- Phase 1: accounts, taxonomy, questions CRUD, practice sessions, analytics aggregates, dashboard endpoint
- Phase 2: exam templates/sessions, review flow, resources listing
- Phase 3: recommendations, notifications, admin workflows
- Phase 4 (optional): billing, advanced analytics, ML recommendations

---

- Implemented a full Django backend plan for the dashboard at `https://open-mrcs-frontend-client.vercel.app/dashboard/`, covering:
  - Apps (`accounts`, `questions`, `practice`, `exams`, `analytics`, `resources`, etc.)
  - Core models and relationships for question bank, sessions, attempts, and aggregates
  - Complete API surface for auth, dashboard, practice, exams, analytics, recommendations, and resources
  - Permissions, caching, observability, and DevOps essentials
