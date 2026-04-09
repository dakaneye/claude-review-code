---
paths:
  - "**/*.py"
---

# Django Quality Rules

## Checklist

### Transaction Discipline
- `[BLOCKER]` Multi-write operations wrapped in `transaction.atomic` — never rely on implicit auto-commit
- `[MAJOR]` Side effects (email, Celery dispatch, webhooks) via `transaction.on_commit`, not inline
- `[MAJOR]` `select_for_update()` for read-modify-write patterns inside transactions

### Security Controls
- `[BLOCKER]` No `@csrf_exempt` on session-authenticated views — use signed webhook verification for legitimate exemptions
- `[BLOCKER]` No `mark_safe()` or `|safe` filter on user-supplied data — use `format_html()` for HTML construction
- `[BLOCKER]` DRF `DEFAULT_PERMISSION_CLASSES` must be `[IsAuthenticated]` or stricter, never `AllowAny`
- `[MAJOR]` Object-level permissions require queryset scoping in addition to `has_object_permission`

### N+1 Prevention
- `[BLOCKER]` `select_related` for ForeignKey/OneToOne accessed in loops or serializers
- `[BLOCKER]` `prefetch_related` for ManyToMany/reverse ForeignKey
- `[MAJOR]` ViewSet `queryset` includes eager loading by default — do not defer to the serializer
- `[MAJOR]` Assert query count with `CaptureQueriesContext` for list endpoints in tests

### Service Layer
- `[MAJOR]` Business logic lives in `services.py` — not in views, serializers, signals, or `save()` overrides
- `[MAJOR]` Signals limited to lifecycle hooks (profile creation, cleanup on delete) — everything else is a service
- `[MAJOR]` Services use keyword-only arguments and return the created/modified object
- `[MINOR]` Read-only complex queries in `selectors.py` if following the Django-styleguide separation

### CI Checks
- `[BLOCKER]` `python manage.py check --deploy` runs in CI — it is the canonical Django security lint
- `[BLOCKER]` `python manage.py makemigrations --check --dry-run` runs in CI — catches uncommitted migrations
- `[MAJOR]` `pip-audit` and `bandit -r . -x tests,migrations` in the CI pipeline

### Testing
- `[MAJOR]` Permission tests assert BOTH the forbidden case (401/403) AND the allowed case
- `[MAJOR]` Use `factory.Sequence` for unique fields — `Faker` collisions cause flaky tests at batch size
- `[MAJOR]` `@pytest.mark.django_db(transaction=True)` when testing `transaction.on_commit` or `select_for_update`

## AI Detection Signals

| Signal | Severity | What to Look For |
|--------|----------|------------------|
| `.save()` in a loop | BLOCKER | Bulk writes as individual saves — use `bulk_create`/`bulk_update`/`QuerySet.update()` |
| Cache without invalidation | BLOCKER | `@cache_page` or low-level cache with no TTL or invalidation story |
| `@csrf_exempt` on regular views | BLOCKER | CSRF suppressed to "make things work" instead of fixing the real issue |
| `mark_safe()` on user input | BLOCKER | XSS vector — use `format_html()` |
| `IsAuthenticated` missing from DRF defaults | BLOCKER | `DEFAULT_PERMISSION_CLASSES = []` or `[AllowAny]` |
| N+1 in list serializers | MAJOR | Related fields accessed in `to_representation` without eager loading |
| Business logic in `save()` override | MAJOR | Side effects in `Model.save()` — belongs in a service |
| Business logic in signals | MAJOR | Complex workflows in `post_save` — belongs in a service |
| Raw SQL for typed queries | MINOR | `cursor.execute()` when ORM + `annotate()` would work |

## Deep Dives

See skills: `django-patterns`, `django-security`, `django-tdd`, `django-verification`
