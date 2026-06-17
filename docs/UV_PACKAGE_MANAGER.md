# UV Package Manager Role

## Purpose
Adds first-class support for the `uv` package manager as an alternative to `pipenv`, `poetry`, and `pip`. Enables fast, deterministic deployments via `uv.lock` and `uv sync`.

## How it works
- Installs uv via the official astral.sh installer, pinned to `uv_version` (default: `0.11.21`).
- Downloads the requested Python interpreter via `uv python find` if `python_version` is set.
- Creates a project-local virtualenv at `{{ virtualenv }}` using `uv venv`.
- Runs `uv sync` from `{{ project_src }}` to install dependencies from `uv.lock`.
- Sets `package_manager_command = {{ uv_binary }} run`, which is consumed by the uvicorn and celery supervisor templates — no template changes required.

## Rationale
- uv is significantly faster than pipenv/poetry (10-100x for cold installs).
- Native `.env` file support (no `poetry-plugin-dotenv` needed).
- Project-local venv layout matches uv's idiomatic design.
- Python version management without pyenv compilation.

## Consequences
- Skips the `pyenv` role entirely when `package_manager == 'uv'` (see `playbooks/platform.yml`).
- The deployed project MUST include a `uv.lock` at the repo root.
- `python_version` is consumed by uv directly via `uv python find <version>`.

## Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `uv_version` | `0.11.21` | uv release version to install |
| `bin_path` | `~/.local/bin` | Where uv is installed |
| `uv_install_path` | `{{ bin_path }}/uv` | Path to uv binary |
| `lang` | `en_US.UTF-8` | Locale for `uv sync` |
| `uv_extra_args` | `""` | Extra args passed to `uv sync` |
| `python_version` | `""` | Python version to install/resolve |
| `virtualenv` | `""` | Path to project-local venv (e.g. `{{ project_root }}/.venv`) |

## Example

```yaml
# group_vars/all.yml
package_manager: 'uv'
python_version: '3.13'
virtualenv: "{{ project_root }}/.venv"
repo:
  url: git@github.com:org/app.git
  version: main
```

## Files

- `roles/uv/defaults/main.yml` — variable defaults
- `roles/uv/tasks/main.yml` — installation and sync tasks
- `roles/uv/tasks/cleanup.yml` — teardown on `coord_delete_all`
- `roles/uv/handlers/main.yml` — reserved for future use
