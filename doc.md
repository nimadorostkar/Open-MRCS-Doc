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
