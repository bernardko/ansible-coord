# UV Package Manager Role

## Purpose
Adds first-class support for the `uv` package manager as an alternative to `pipenv`, `poetry`, and `pip`. Enables fast, deterministic deployments via `uv.lock` and `uv sync`.

## How it works
- Installs uv via the official astral.sh installer, pinned to `uv_version` (default: `0.11.21`).
- If `python_version` is set, runs `uv python install {{ python_version }}` to provision the requested interpreter, then `uv python find {{ python_version }}` to resolve its absolute path. This path becomes `python_interpreter` and is used both to build the venv and as a sanity check that the host actually has the requested Python available.
- Inspects the existing virtualenv at `{{ virtualenv }}`. If `uv_archive_legacy_venv` (default `true`) is set **and** the existing venv is missing a `uv = ...` line in `pyvenv.cfg` **or** its `version_info` does not match the requested Python's major.minor, the venv is moved aside to `{{ virtualenv }}.legacy-YYYY-MM-DD` so it is rebuilt cleanly.
- Creates a project-local virtualenv at `{{ virtualenv }}` with `uv venv --python {{ python_interpreter }}`.
- Runs `uv sync` from `{{ project_src }}` to install dependencies from `uv.lock`. Failures are surfaced with the captured stdout/stderr — the role no longer silently succeeds.
- Sets `package_manager_command = {{ uv_binary }} run`, which is consumed by the uvicorn and celery supervisor templates — no template changes required.

## Python resolution
- `python_version` is the source of truth. The role will install it if missing and fail the play if `uv` cannot resolve it.
- The previous fallback to `{{ virtualenv }}/bin/python3` is preserved only as a last-resort when the existing venv already has a working `python3`; it is not used to *create* a new venv. The historical "silently fall back to a different Python" behavior is gone.
- To restore the legacy fallback semantics (e.g. when you genuinely want to use whatever Python is on PATH), set `uv_allow_python_fallback: true`. The role will then fall back to `{{ ansible_python_interpreter | default('/usr/bin/python3') }}`.

## Migrating from pip/pipenv
- The first deploy after switching a project to `uv` will usually have a legacy venv (no `uv` marker in `pyvenv.cfg`) at `{{ virtualenv }}`. With `uv_archive_legacy_venv: true` (the new default), the role will:
  1. Detect the legacy venv during the `uv | Inspect existing venv metadata` step.
  2. Move it to `{{ virtualenv }}.legacy-YYYY-MM-DD` (preserved for rollback, not deleted).
  3. Build a fresh venv with `uv venv` and `uv sync` the locked dependencies.
- No manual `rm -rf .venv` is required. If you want to keep the legacy venv untouched and let `uv sync` reconcile into it instead, set `uv_archive_legacy_venv: false`.

## Rationale
- uv is significantly faster than pipenv/poetry (10-100x for cold installs).
- Native `.env` file support (no `poetry-plugin-dotenv` needed).
- Project-local venv layout matches uv's idiomatic design.
- Python version management without pyenv compilation.

## Consequences
- Skips the `pyenv` role entirely when `package_manager == 'uv'` (see `playbooks/platform.yml`).
- The deployed project MUST include a `uv.lock` at the repo root.
- `python_version` is consumed by uv directly via `uv python install` + `uv python find`. A Python that uv cannot install (or that is not present) is now a hard failure rather than a silent fallback.
- A legacy venv from pip/pipenv will be archived on the first deploy after switching to `uv` (when `uv_archive_legacy_venv: true`). The archived copy is left in place for rollback.

## Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `uv_version` | `0.11.21` | uv release version to install |
| `bin_path` | `~/.local/bin` | Where uv is installed |
| `uv_install_path` | `{{ bin_path }}/uv` | Path to uv binary |
| `lang` | `en_US.UTF-8` | Locale for `uv sync` |
| `uv_extra_args` | `""` | Extra args passed to `uv sync` |
| `python_version` | `""` | Python version to install/resolve |
| `uv_allow_python_fallback` | `false` | When true, fall back to the system Python if the requested version cannot be resolved |
| `uv_archive_legacy_venv` | `true` | When true, archive a non-uv venv or a venv with a mismatched Python to `{{ virtualenv }}.legacy-YYYY-MM-DD` and rebuild |
| `virtualenv` | `""` | Path to project-local venv (e.g. `{{ project_root }}/.venv`) |

## Example

```yaml
# group_vars/all.yml
package_manager: 'uv'
python_version: '3.14.6'
virtualenv: "{{ project_root }}/.venv"
repo:
  url: git@github.com:org/app.git
  version: main
```

## Files

- `roles/uv/defaults/main.yml` — variable defaults
- `roles/uv/tasks/main.yml` — installation, resolve, archive, and sync tasks
- `roles/uv/tasks/cleanup.yml` — teardown on `coord_delete_all`
- `roles/uv/handlers/main.yml` — reserved for future use
