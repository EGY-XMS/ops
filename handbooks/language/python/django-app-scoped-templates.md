# Handbook: Django App-Scoped Templates

**Scope:** PRs touching `**/*.py` or `**/templates/**` (handbook lives under `language/python/` — Rex loads it on Python diffs).
**Enforcement:** blocking.

## The rule

Every Django template must live inside the app that owns it, namespaced by the app name:

```
<app>/templates/<app>/<template>.html
```

**Correct:**
```
water_quality/templates/water_quality/dashboard.html
audit/templates/audit/object_history.html
two_factor/templates/two_factor/otp_verify.html
```

**Wrong — never do these:**
```
templates/dashboard.html              # project-level templates/ root
templates/water_quality/dashboard.html  # project-level root even with namespace dir
<app>/templates/<template>.html       # missing the app namespace subdirectory
```

## Why

Django's template loader searches all installed apps in order. Without the `<app>/` subdirectory inside `templates/`, a template named `dashboard.html` in one app silently shadows a template with the same name in another app — whichever app appears first in `INSTALLED_APPS` wins. This produces environment-dependent rendering bugs that are invisible in development and only surface when app order changes or a new app is added.

The `<app>/templates/<app>/` double-layer is Django's canonical solution. `{% extends "audit/object_history.html" %}` is unambiguous regardless of installed-app order.

## What Rex flags

- Any new template file whose path does not match `<app>/templates/<app>/**`
- Any new template file placed directly under a project-level `templates/` directory at the repo root
- Any `{% include %}` or `{% extends %}` reference that omits the app-namespace prefix (e.g. `{% extends "base.html" %}` instead of `{% extends "myapp/base.html" %}`)
- Any addition to `TEMPLATES[0]['DIRS']` that points to a project-level `templates/` directory unless accompanied by an explicit comment explaining why app-scoped placement is not possible

## Exceptions

- Third-party packages that inject their own template directories (e.g. `django-unfold`, `django-allauth`) — these are not under your control; do not flag.
- `registration/` templates required by Django's built-in auth views at the `registration/` path — flag with a `suggestion:` to consider overriding via an app-scoped template instead.
