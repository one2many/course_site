# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Open Visualization Academy (OVA) — a Django 5.2 / Wagtail 7.0 online course platform. Users log in via email one-time codes (django-allauth), watch Vimeo-hosted video segments, take quizzes, track progress, and earn PDF certificates. Deployed to Azure Web App with PostgreSQL, Azure Blob Storage for media, and Daphne as the ASGI server.

## Commands

```bash
# Install dependencies (uses uv with pyproject.toml + uv.lock)
uv pip install --system -r <(uv pip compile pyproject.toml)
# Or in a venv:
uv sync

# Run dev server (Daphne via ASGI)
python manage.py runserver              # Django dev server (simpler)
daphne -b 0.0.0.0 -p 8000 ova.asgi:application  # ASGI (production-like)

# Docker dev
docker compose up --build               # builds and starts ova_dev container on :8000

# Database
python manage.py migrate
python manage.py createsuperuser

# Static files
python manage.py collectstatic --noinput

# Tests (pytest with pytest-django; conftest uses in-memory SQLite)
pytest                                   # run all tests
pytest courses/tests/test_progress_completion.py  # single file
pytest -k test_progress_completion_contract_single_chapter  # single test

# Linting / formatting
ruff check .                             # linter
ruff check . --fix                       # auto-fix
black .                                  # formatter

# Management commands
python manage.py import_course_structure path/to/structure.json --dry-run
python manage.py cleanup_unverified_users
```

## Architecture

### Django Project: `ova/`
The Django project is named `ova`. Settings are split: `ova/settings/base.py` (shared), `dev.py` (DEBUG=True, debug-toolbar, django-extensions), `production.py` (secure cookies, logging). `manage.py` defaults to `ova.settings.dev`. Templates live in `ova/templates/` (base.html, account/, admin/, partials/). Static assets are in `ova/static/` with CSS/JS organized into `imports/` modules.

### Apps

**`courses`** — Core app. Wagtail page hierarchy: `CoursesIndexPage` → `CoursePage` → `ChapterPage` → `SegmentPage`. Progress models: `SegmentProgress` (percent_watched), `ChapterProgress`, `CourseProgress`, `QuizProgress`. Quiz models: `Quiz` → `Question` → `Choice` (inline via ParentalKey/ClusterableModel). `QuizMixin` on SegmentPage handles quiz submission/grading. The `views.py` has two API endpoints under `/api/`: `update_progress` (POST, tracks video watching and cascades chapter/course completion) and `generate_certificate` (POST, calls Azure Function to render HTML→PDF).

**`users`** — Custom User model (email-based, no username). Auto-creates users on first login code request (`AutoCreateLoginView`). Custom allauth adapter (`ACSAccountAdapter`) sends emails via Azure Communication Services.

**`home`** — HomePage plus static pages (About, Sponsors, Accessibility, Brand).

### Key Patterns

- **Wagtail page tree**: Content is managed via Wagtail's page tree. CoursePage.serve() redirects to the first segment; CoursePage.get_url() resolves to first segment URL. All page types auto-generate slugs from titles on save.
- **Coming Soon access control**: CoursePage.coming_soon field gates access. Checked in serve() of CoursePage, ChapterPage, and SegmentPage. Allowed emails stored as comma-separated text.
- **Progress cascade**: When a segment reaches 100%, `_is_chapter_complete` checks all sibling segments; when all chapters complete, course is marked complete. Completion is monotonic (never reverts).
- **Auth flow**: Passwordless login via allauth one-time codes. Users are auto-created (active) when they request a code. `RememberMeLoginForm` preserves session preference across the code verification step.
- **Admin analytics**: `ova/admin.py` monkey-patches Django admin with `/django-ova-admin/analytics/` dashboard showing user, engagement, and content metrics.

### URL Structure
- `/` — Wagtail pages (home, courses, chapters, segments)
- `/api/progress/update/` — video progress tracking
- `/api/courses/<id>/certificate/` — certificate generation
- `/ova-admin/` — Wagtail admin
- `/django-ova-admin/` — Django admin (with analytics dashboard)
- `/accounts/` — allauth authentication

### Environment
Requires a `.env` file (see `env-template`). Key vars: `DATABASE_URL`, `DB_ENGINE`, `DJANGO_SECRET_KEY`, `AZURE_ACCOUNT_NAME`, `AZURE_CONTAINER_MEDIA`, `AZURE_STORAGE_CONNECTION`, `AZURE_COMM_EMAIL_STRING`, `CERT_FUNCTION_URL`.

### Testing
pytest.ini sets `DJANGO_SETTINGS_MODULE = ova.settings.production` but `courses/tests/conftest.py` overrides the database to in-memory SQLite. Tests use `pytest-django` and `wagtail-factories`.
