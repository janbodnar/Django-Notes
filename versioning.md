# Versioning, Releases, and Project Lifecycle Best Practices for Django Projects

Maintaining a predictable, reproducible Django project over time requires  
deliberate practices around versioning, dependency management, database  
migrations, branching, and deployment. This document provides actionable  
guidance for intermediate Django developers and small teams who want stable,  
maintainable projects from day one through long-term production operation.  


## Why Versioning Matters in Django Projects

Versioning is not just about labelling releases — it is the backbone of  
reproducibility, debugging, and long-term stability.  

**Reproducibility**  
A pinned, tagged codebase combined with a locked dependency file lets any  
team member (or CI server) recreate an identical environment from scratch.  
Without version tags and locked dependencies you cannot guarantee that  
`pip install -r requirements.txt` produces the same result tomorrow as it  
does today.  

**Debugging and rollback**  
When an incident occurs in production, a Git tag tied to the deployed  
artifact (Docker image, wheel, or tarball) lets you:  

- Reproduce the exact code that was running.  
- Diff the faulty release against the previous one.  
- Roll back by redeploying the previous tagged artifact.  

**Dependency stability**  
Django's ecosystem moves fast. An unversioned project silently picks up  
new library versions on every fresh install, which can introduce breaking  
changes with no audit trail. Explicit version constraints make upgrades a  
deliberate, reviewable decision rather than an invisible side-effect.  


## Semantic Versioning for Django Apps

Semantic versioning (`MAJOR.MINOR.PATCH`) gives consumers and teammates  
a predictable signal about the impact of each release.  

**MAJOR.MINOR.PATCH explained**  

| Segment | When to increment | Example trigger |
|---------|-------------------|-----------------|
| `MAJOR` | Breaking public API or behaviour change | Removed endpoint, changed URL structure |
| `MINOR` | New backward-compatible feature | New API endpoint, new model field |
| `PATCH` | Backward-compatible bug fix | Fixed 500 error, corrected validation |

**When to bump each number**  

- Bump `PATCH` for every bug-fix release that ships no new features.  
- Bump `MINOR` whenever you add functionality that does not break existing  
  clients or data structures.  
- Bump `MAJOR` for any change that requires consumers to update their  
  integrations or that removes/renames public API surface.  

**Example version progression**  

```
1.0.0  — initial production release
1.0.1  — hotfix: fixed null-pointer in checkout view
1.1.0  — added product recommendations endpoint
1.2.0  — added order export to CSV
2.0.0  — replaced session auth with JWT; broke existing auth cookie clients
```

**How Django's own release cycle influences your project**  

Django follows a time-based release cycle with a new feature release  
approximately every eight months and Long-Term Support (LTS) releases  
every two years. You should:  

- Target a Django LTS release for new projects to maximise the support  
  window (three years of security patches).  
- Align your own `MAJOR` version bumps with Django `MAJOR` upgrades so  
  your changelog clearly signals the framework requirement change.  
- Check the [Django release roadmap](https://www.djangoproject.com/download/)  
  before planning a `MINOR` release that will require a Django upgrade.  


## Managing Dependencies & Python Environment

**Pinning vs ranges**  

| Approach | Example | Use when |
|----------|---------|----------|
| Exact pin | `Django==5.0.4` | Reproducible production builds |
| Compatible release | `Django~=5.0` | Library packages that must support a range |
| Minimum bound | `Django>=4.2` | Rarely recommended; risks drift |

For applications (as opposed to reusable libraries) always use exact pins  
in your lock file so every environment installs identical wheels.  

**Using `requirements.txt`, `pyproject.toml`, or lock files**  

The recommended workflow with `uv`:  

```bash
# Create project and virtual environment
uv init myproject
cd myproject
uv venv
source .venv/bin/activate

# Add runtime dependencies (uv writes exact pins to uv.lock)
uv add django==5.0.4
uv add psycopg[binary]==3.1.19
uv add gunicorn==22.0.0

# Add dev/test dependencies
uv add --dev pytest-django==4.8.0
uv add --dev coverage==7.4.4
```

Commit both `pyproject.toml` and `uv.lock` to version control. The lock  
file is the source of truth for reproducible installs.  

For projects that still use `pip`, generate a pinned requirements file:  

```bash
pip install -r requirements.in       # abstract requirements
pip freeze > requirements.txt        # pinned lock file
```

Keep a `requirements.in` (or `pyproject.toml` with abstract ranges) for  
human-readable intent and a `requirements.txt` for reproducible installs.  

**Handling Django upgrades safely**  

1. Create a dedicated upgrade branch: `git checkout -b upgrade/django-5.1`  
2. Update the Django version in `pyproject.toml` or `requirements.in`.  
3. Run `uv lock` (or `pip-compile`) to regenerate the lock file.  
4. Run `python manage.py check` and fix any reported issues.  
5. Run the full test suite; address failures before merging.  
6. Review Django's release notes for your upgrade path.  

**Dependency update workflow**  

Run dependency audits on a schedule — at minimum monthly:  

```bash
# Check for known vulnerabilities
pip install pip-audit
pip-audit

# List outdated packages
pip list --outdated

# Update a single package safely
uv add django==5.1.2
uv lock
```

Never update all dependencies at once. Update one package at a time,  
run tests, and commit each update separately so failures are easy  
to bisect.  


## Database Migrations & Versioning

**Migration hygiene**  

- Name migrations descriptively using `--name`:  

  ```bash
  python manage.py makemigrations --name add_published_field_to_post
  ```

- Keep migrations small and focused on a single logical change.  
- Never edit a migration that has already been applied in production.  
- Store migrations in version control alongside the code that requires  
  them.  

**Avoiding migration conflicts**  

Conflicts arise when two branches both create migrations with the same  
sequence number. Prevent this with a team convention:  

- Always pull and merge `main` into a feature branch before running  
  `makemigrations`.  
- Use `python manage.py showmigrations` to verify the current state  
  before creating new migrations.  
- Resolve conflicts with:  

  ```bash
  python manage.py makemigrations --merge
  ```

  Review the generated merge migration before committing.  

**Squashing migrations**  

After an app accumulates dozens of migrations, squash them to improve  
startup performance and readability:  

```bash
python manage.py squashmigrations myapp 0001 0040
```

The squashed migration retains all `replaces` references so existing  
environments apply it correctly. Remove the original migrations only  
after the squashed migration has been applied everywhere.  

**Safe migration deployment order**  

Always deploy migrations *before* deploying the code that depends on  
them. The recommended sequence for a zero-downtime deploy:  

1. Deploy the migration on the database (add new columns, create new  
   tables).  
2. Deploy the new code that reads the new schema.  
3. If removing old columns, first deploy a code version that no longer  
   reads them, then deploy the migration that drops them.  

This backward-compatible approach keeps the old code version functional  
while the new version rolls out.  

**Handling breaking schema changes**  

Rename a column in three steps instead of one:  

```
Step 1: Add the new column; copy data in a data migration.
Step 2: Deploy code that writes to both columns; reads from the new one.
Step 3: Drop the old column once all instances run the new code.
```

Never rename a column in a single migration on a live system; it will  
cause errors on the old code that is still running during the deploy.  


## Release Workflow & Branching Strategy

**Git tags for releases**  

Tag every production release with an annotated tag:  

```bash
git tag -a v1.2.0 -m "Release 1.2.0: add order export"
git push origin v1.2.0
```

Annotated tags store the tagger identity and timestamp, making them  
more useful than lightweight tags for release audit trails.  

**Recommended branching models**  

For most small-to-medium Django projects, a simplified trunk-based model  
works well:  

```
main          — always deployable; protected branch
feature/*     — short-lived; merged back to main via PR
hotfix/*      — branched from main; merged back and tagged
release/*     — optional; used for staging stabilisation
```

- Feature branches should live no longer than a few days to avoid  
  large, conflict-prone merges.  
- All merges to `main` require a passing CI check and code review.  

**Release branches vs hotfix branches**  

| Type | Branched from | Merges into | Purpose |
|------|--------------|-------------|---------|
| `release/1.2.0` | `main` | `main` | Final QA, version bump, changelog |
| `hotfix/1.2.1` | `main` (or the release tag) | `main` | Critical production fix |

After merging a hotfix, always tag the result immediately:  

```bash
git checkout main
git merge --no-ff hotfix/1.2.1
git tag -a v1.2.1 -m "Hotfix 1.2.1: fix payment race condition"
git push origin main v1.2.1
```

**Automated CI checks**  

Every PR to `main` should run at minimum:  

```yaml
# .github/workflows/ci.yml (GitHub Actions example)
jobs:
  test:
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: uv sync --frozen
      - run: python manage.py check --deploy
      - run: python manage.py migrate --run-syncdb
      - run: pytest --cov=. --cov-fail-under=80
```

Block merges on failing checks; never bypass them for "small" changes.  


## Deployment Versioning

**Tagging Docker images**  

Every image pushed to a registry should carry at least two tags:  

```bash
IMAGE=registry.example.com/myapp

git_tag=$(git describe --tags --abbrev=0)   # e.g. v1.2.0
git_sha=$(git rev-parse --short HEAD)        # e.g. a3f9c12

docker build \
  -t $IMAGE:$git_tag \
  -t $IMAGE:$git_sha \
  -t $IMAGE:latest \
  .

docker push $IMAGE:$git_tag
docker push $IMAGE:$git_sha
docker push $IMAGE:latest
```

Never deploy an image tagged only as `latest` in production — `latest`  
is mutable. Always reference the immutable SHA or semver tag in your  
deployment manifests.  

**Tracking deployed versions**  

Expose the current version through a health endpoint so monitoring  
tools and deployment scripts can confirm which version is live:  

```python
# myapp/views.py
import os
from django.http import JsonResponse

def health(request):
    return JsonResponse({
        "status": "ok",
        "version": os.environ.get("APP_VERSION", "unknown"),
        "commit": os.environ.get("GIT_COMMIT", "unknown"),
    })
```

Set `APP_VERSION` and `GIT_COMMIT` as environment variables in your  
container runtime or deployment pipeline.  

**Rollback strategy**  

A reliable rollback requires:  

1. The previous image tag still available in the registry.  
2. No irreversible migration (no dropped columns, no renamed tables).  
3. A documented procedure:  

```bash
# Kubernetes example
kubectl set image deployment/myapp \
  myapp=registry.example.com/myapp:v1.1.3

# Verify rollout
kubectl rollout status deployment/myapp
```

If a migration cannot be reversed cleanly, the rollback plan must  
include a manual data-repair step. Document this in the release notes  
before shipping.  

**Zero-downtime migration workflow**  

For rolling deploys with multiple application instances:  

1. Deploy migration-only: run `python manage.py migrate` before  
   updating any application pod.  
2. The migration must be backward-compatible with the currently running  
   code version (no destructive changes).  
3. Roll out new application pods one at a time with health checks  
   between each.  
4. Verify the health endpoint reports the new version on all pods  
   before closing the deployment window.  


## Handling Django Upgrades

**LTS vs non-LTS**  

Django publishes two release types:  

| Type | Support window | Example |
|------|---------------|---------|
| LTS  | ~3 years (security fixes until EOL) | Django 4.2, 5.2 |
| Non-LTS | ~16 months | Django 5.0, 5.1 |

Use LTS releases for production systems where stability and a long  
support window matter. Track the [EOL dates](https://www.djangoproject.com/download/#supported-versions)  
and plan upgrades at least six months before EOL to avoid running on  
an unsupported version.  

**Reading release notes**  

Before upgrading, read the full release notes for every version you  
are skipping. Pay particular attention to:  

- *Backwards incompatible changes* — these require code changes.  
- *Deprecated features* — these will break in a future version.  
- *New features* — opportunities to remove third-party workarounds.  

**Deprecation warnings**  

Run your test suite with warnings treated as errors to catch  
deprecations early:  

```bash
python -W error::DeprecationWarning -W error::PendingDeprecationWarning \
  manage.py test
```

Or in `pytest.ini`:  

```ini
[pytest]
filterwarnings =
    error::DeprecationWarning
    error::PendingDeprecationWarning
```

Fix all deprecation warnings before upgrading to the next major  
Django version, where those warnings become hard errors.  

**Testing upgrade paths**  

1. Create an upgrade branch targeting the new Django version.  
2. Update `pyproject.toml` or `requirements.in` with the new version.  
3. Regenerate the lock file and install dependencies.  
4. Run `python manage.py check`.  
5. Run the full test suite; fix failures.  
6. Run the development server and perform manual smoke tests.  
7. Test the migration path on a copy of the production database if  
   the upgrade includes migration squashing or schema changes.  


## Configuration & Settings Versioning

**Environment-specific settings**  

Avoid a single `settings.py` that branches on environment flags.  
Instead, split settings into a package:  

```
mysite/settings/
    __init__.py     # empty or imports base
    base.py         # shared across all environments
    development.py  # overrides for local dev
    production.py   # overrides for production
    test.py         # overrides for test runner
```

Point each environment at the right module:  

```bash
# .env or shell
DJANGO_SETTINGS_MODULE=mysite.settings.production
```

**Secrets management**  

Never commit secrets to version control. Use environment variables  
loaded via `python-decouple` or a secrets manager:  

```python
# mysite/settings/base.py
from decouple import config

SECRET_KEY = config("SECRET_KEY")
DATABASE_URL = config("DATABASE_URL")
```

For production, pull secrets from a vault (AWS Secrets Manager,  
HashiCorp Vault, or similar) rather than `.env` files on disk.  

**Avoiding drift between environments**  

Environment drift — where staging and production have different  
settings, package versions, or database states — is a leading cause  
of "works on staging, fails in production" incidents. Prevent it by:  

- Building a single Docker image and promoting it through environments  
  (do not rebuild per environment).  
- Using the same `uv.lock` / `requirements.txt` in every environment.  
- Running the same migration command on every environment as part of  
  the deployment pipeline.  
- Diffing environment variables between staging and production as part  
  of release checklists.  


## Documentation & Changelog Practices

**Writing meaningful changelogs**  

Follow the [Keep a Changelog](https://keepachangelog.com/) format.  
Every release section should list changes under:  

```markdown
## [1.2.0] - 2026-04-30

### Added
- Order export to CSV via `/api/orders/export/`.

### Changed
- Switched from `Pillow` to `pillow-simd` for faster image resizing.

### Fixed
- Race condition in payment confirmation webhook handler.

### Deprecated
- `GET /api/v1/products/` is deprecated; use `/api/v2/products/`.

### Removed
- Removed legacy XML export endpoint `/api/orders/export.xml`.

### Security
- Upgraded `cryptography` to 42.0.5 (CVE-2024-26130).
```

**Documenting breaking changes**  

Breaking changes must appear in three places:  

1. The `CHANGELOG.md` `Removed` or `Changed` section with a migration  
   guide.  
2. The Git tag annotation (`git tag -a v2.0.0 -m "..."`).  
3. A deprecation notice in the previous release, giving consumers  
   time to adapt.  

**Linking releases to issues/PRs**  

Reference issue and PR numbers in changelog entries and tag  
annotations. On GitHub, this auto-links in the releases page:  

```markdown
### Fixed
- Race condition in payment webhook handler (#412, PR #418).
```

Use `git log v1.1.0..v1.2.0 --oneline` to generate a draft list of  
commits to summarise into the changelog.  


## Common Pitfalls to Avoid

**Unpinned dependencies**  
Running `pip install django` without a version constraint will install  
the latest release on every fresh environment. Use a lock file and  
pin exact versions.  

**Skipping migrations**  
Never delete or skip migrations to resolve a conflict. Always use  
`makemigrations --merge`. Skipped migrations leave the schema in an  
unknown state that is difficult to recover from.  

**Mixing feature work with release work**  
A release branch should contain only version bump commits, changelog  
updates, and last-minute bug fixes. Merging unreviewed feature work  
into a release branch bypasses the normal review process and  
introduces unplanned risk.  

**Ignoring deprecation warnings**  
Deprecation warnings are advance notice of future breaking changes.  
Treat them as bugs and fix them immediately; do not accumulate a  
backlog that will block the next Django upgrade.  

**Deploying without tagging**  
If you deploy a commit that carries no tag, you lose the ability to  
reproduce that exact deployment or to clearly communicate "what is  
running in production." Tag every production deploy, even hotfixes.  

**Applying destructive migrations before updating code**  
Dropping or renaming a column while the old code is still running  
will cause immediate errors. Always ensure the old code no longer  
references a schema element before removing it.  

**Running different code versions across instances during a deploy**  
During a rolling deploy, two code versions run simultaneously. Write  
migrations that are compatible with both the old and new code, and  
only remove backward-compatibility shims in a follow-up release.  
