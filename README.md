## OpenMRCS Backend Technical Specification (Django)

Reference UI: [OpenMRCS Dashboard](https://open-mrcs-frontend-client.vercel.app/dashboard/)

### Scope
Design and implement a Django-based backend for the OpenMRCS platform to power:
- Authentication and user profiles
- Practice Mode (category-targeted practice with explanations)
- Exam Mode (timed mock exams, Paper 1/2)
- Question bank with taxonomy
- Dashboard aggregates (progress, performance, study time, recent activity)
- Suggested topics and high-yield resources
- Notifications and optional billing
- Admin/CMS, analytics, and observability

### Architecture

- Framework: Django + Django REST Framework (DRF)
- DB: PostgreSQL
- Cache/Queue: Redis (cache + Celery broker)
- Async: Celery + Celery Beat (scheduled jobs)
- Storage: S3-compatible for media (or external URLs)
- Auth: JWT (SimpleJWT), optional social auth
- API: REST, versioned: /api/v1/...
- Admin/CMS: Django Admin; Wagtail optional for richer content
- Observability: Sentry + OpenTelemetry
- Security: DRF throttling, IP rate limits, object-level permissions

### Django Apps

- accounts: users, profiles, roles, settings, GDPR
- taxonomy: categories, subcategories
- questions: question bank, answer options, media, explanations
- practice: practice sessions/attempts, scoring
- exams: exam templates, sessions, attempts, scoring
- analytics: aggregates, progress, study time
- recommendations: suggested topics, rules/ML
- resources: videos, seminars, articles
- notifications: in-app/user messages
- billing (optional): plans, subscriptions, entitlements
- cms: static/legal pages, featured sections
- audit: event log, staff changes, exports

### Data Model (Core)

```python
from django.contrib.auth.models import AbstractUser
from django.db import models

# accounts
class User(AbstractUser):
    email = models.EmailField(unique=True)
    is_email_verified = models.BooleanField(default=False)
    role = models.CharField(max_length=20, default="user")  # user/editor/examiner/admin

class Profile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE, related_name="profile")
    display_name = models.CharField(max_length=120, blank=True)
    avatar = models.URLField(blank=True)
    timezone = models.CharField(max_length=64, default="UTC")
    prefers_explanations = models.BooleanField(default=True)

# taxonomy
class Category(models.Model):
    name = models.CharField(max_length=120, unique=True)
    slug = models.SlugField(unique=True)
    description = models.TextField(blank=True)
    order = models.PositiveIntegerField(default=0)

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
    reference = models.TextField(blank=True)
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
    kind = models.CharField(max_length=20, default="image")  # image/audio/video

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
    url = models.URLField()
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

Indexes and constraints:
- Unique: `User.email`, `Category.slug`
- FKs indexed; partial indexes for `Question(is_active)` and `Question(category)`
- Add composite index: `CategoryAggregate(user, category)`

### API Design (v1)

Base: `/api/v1/...` (JWT Bearer; some public endpoints)

#### Auth & Profile
- POST `/auth/register`
- POST `/auth/login` → `{ access, refresh }`
- POST `/auth/refresh`
- POST `/auth/logout`
- POST `/auth/forgot-password`, POST `/auth/reset-password`
- GET `/me`, PATCH `/me`
- GET `/me/notifications`, POST `/me/notifications/{id}/read`

Example:
```json
// POST /auth/login
{ "email": "user@example.com", "password": "secret" }
// 200
{ "access": "jwt...", "refresh": "jwt..." }
```

#### Dashboard (single fetch per UI)
- GET `/dashboard`
  - overall_progress: `{ completed, total }`
  - performance: `{ avg_score_percent, correct, incorrect }`
  - study_time: `{ hours_this_week, delta_vs_last_week }`
  - recent_activity: `{ last_practice, last_exam }`
  - suggested_topics: list
  - featured_resources: list

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

#### Taxonomy & Questions
- GET `/categories`
- GET `/categories/{slug}`
- GET `/questions?category=...&subcategory=...&difficulty=...`
- GET `/questions/{id}`
- Staff: POST `/questions`, PUT/PATCH `/questions/{id}`, DELETE `/questions/{id}`, POST `/questions/import` (CSV/JSON)

#### Practice Mode
- POST `/practice/sessions` → `{ categories, subcategories, question_count, include_explanations }`
- GET `/practice/sessions/{id}`
- POST `/practice/sessions/{id}/attempts`
  - request: `{ question_id, selected_option_ids, time_spent_sec }`
  - response: `{ is_correct, correct_option_ids, explanation? }`
- POST `/practice/sessions/{id}/complete`
- GET `/practice/sessions` (history)
- GET `/practice/summary` (per-category aggregates)

#### Exam Mode
- GET `/exams/templates`
- POST `/exams/sessions` → `{ template_id }`
- GET `/exams/sessions/{id}`
- POST `/exams/sessions/{id}/attempts` (per question or batched)
- POST `/exams/sessions/{id}/submit`
- GET `/exams/sessions` (history)
- GET `/exams/sessions/{id}/review`

#### Analytics
- GET `/analytics/overview`
- GET `/analytics/study-time?range=week|month`
- GET `/analytics/categories`
- GET `/analytics/heatmap`
- GET `/analytics/trends`

#### Recommendations
- GET `/recommendations/topics`
- POST `/recommendations/dismiss` (optional)

#### Resources
- GET `/resources?featured=true`
- GET `/resources?category={slug}`
- GET `/resources/{id}`
- Staff CRUD

#### Notifications
- GET `/notifications`
- POST `/notifications/{id}/read`
- Staff broadcast/targeted create

#### Billing (optional)
- GET `/billing/plans`
- POST `/billing/checkout`
- POST `/billing/webhooks/{provider}`
- GET `/billing/subscription`

### Business Logic

- Practice attempt:
  - Correctness = set equality of `selected_option_ids` and correct option IDs
  - Update `DailyStudyStat` and `CategoryAggregate`
- Exam submission:
  - Lock session; compute score; sum `time_spent_sec`
- Progress completion:
  - Configurable rule, e.g., answered ≥ N and score ≥ X per category
- Study time:
  - Derived from attempts; optional heartbeats to track on-page time
- Suggestions:
  - Categories with low score and enough attempts; also “Not started” topics

### Permissions & Roles

- Roles: user, editor (content), examiner (exam templates), admin
- DRF permissions:
  - Users access only their data
  - Editors manage `Question`, `Resource`
  - Examiners manage `ExamTemplate`
  - Admin full access
- Object-level checks enforced

### Performance & Scale

- Caching: per-user `/dashboard` (TTL 30–60s)
- Denormalization: `CategoryAggregate`
- Query tuning: `select_related`, `prefetch_related`
- Indexes on common filters; partial indexes where useful
- Rate limiting for auth and attempt submissions
- Bulk import via Celery with validation reports

### Admin & CMS

- Django Admin for all models with search/filters
- Rich text for explanations/resources
- Draft/published workflow; audit trail
- Legal pages in `cms` (Privacy, Terms)

### Observability & Quality

- Sentry for exceptions
- OpenTelemetry tracing on hot endpoints
- Structured logging with request/user/session IDs
- Tests:
  - Unit: scoring, aggregates
  - API: auth, practice, exams
  - Property tests for correctness logic
- Seed scripts: categories (≈25), sample questions, exams, resources

### Environment & DevOps

- Envs: dev/staging/prod with separate secrets
- Migrations & seed commands
- CI: tests, lint, mypy
- CD: run migrations, collectstatic; warm caches
- Backups: daily PG dumps, retention policy
- Feature flags for gating modules (billing, ML)

### Frontend Integration Mapping (Dashboard)

Source: [OpenMRCS Dashboard](https://open-mrcs-frontend-client.vercel.app/dashboard/)

- Overall Progress → GET `/dashboard.overall_progress`
- Performance (avg score, correct/incorrect) → GET `/dashboard.performance`
- Study Time (this week + delta) → GET `/dashboard.study_time`
- Recent Activity (last practice, last exam) → GET `/dashboard.recent_activity`
- Suggested Topics (low score, not started) → GET `/dashboard.suggested_topics`
- High Yield Resources (featured) → GET `/resources?featured=true`
- Practice Mode → POST `/practice/sessions` → submit attempts → complete
- Exam Mode → POST `/exams/sessions` → submit → review

### Non-Functional Requirements

- Availability: 99.9%
- Latency: P50 < 200ms, P95 < 600ms for `/dashboard`
- Security: OWASP ASVS L2; JWT rotation; email verification
- Privacy: GDPR export/delete; data minimization
- Accessibility: resource metadata supports captions/alt text

### Rollout Plan

- Phase 1: accounts, taxonomy, question CRUD, practice sessions, analytics aggregates, `/dashboard`
- Phase 2: exams (templates, sessions, review), resources
- Phase 3: recommendations, notifications, admin workflows
- Phase 4 (opt): billing, advanced analytics, ML recommendations

### Example Payloads

```json
// GET /categories
[
  { "slug": "anatomy-embryology", "name": "Anatomy & Embryology",
    "progress": { "answered": 320, "correct": 172, "score_percent": 53.75 } },
  { "slug": "orthopaedic-surgery", "name": "Orthopaedic Surgery",
    "progress": { "answered": 110, "correct": 69, "score_percent": 62.73 } }
]

// GET /exams/templates
[
  { "id": 1, "name": "Exam A, Paper 2", "paper": "Paper 2",
    "duration_minutes": 180, "total_questions": 180, "description": "" }
]

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

### ER Relationships (Textual)

- User 1–1 Profile
- Category 1–N Subcategory
- Category/Subcategory 1–N Question
- Question 1–N AnswerOption, 1–N QuestionMedia
- User 1–N PracticeSession 1–N PracticeAttempt
- User 1–N ExamSession 1–N ExamAttempt
- User 1–N DailyStudyStat
- User 1–N CategoryAggregate (per Category)
- User 1–N SuggestedTopic (per Category)
- Category 1–N Resource (nullable)
- User 1–N Notification
- User 1–N Subscription (optional)

### Security & Compliance

- Short-lived access tokens, refresh rotation, device revocation
- Email verification; password policy; 2FA optional
- DRF throttling; IP-based rate limits
- GDPR: GET `/me/export`, DELETE `/me`
- Audit log for staff actions and session mutations

### References

- Dashboard UI basis: [OpenMRCS Dashboard](https://open-mrcs-frontend-client.vercel.app/dashboard/)
