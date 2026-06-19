# Application Role

Orchestrates the full application lifecycle on a server: cloning/updating the repo, configuring the runtime environment, managing virtualenvs, running service setup roles, and deploying/updating services.

## Task Flow

The role runs tasks in this approximate order:

1. **Git** — Clone or pull the application repository.
2. **Environment** — Install `python-dotenv` (pip-based installs only), create `.env` file from template, upload additional template files.
3. **Package Manager** — Delegates to the `package_manager` role to set up the virtualenv (pipenv, pip, or uv).
4. **Django** (if defined) — Runs Django-specific setup.
5. **FastAPI** (if defined) — Runs FastAPI-specific setup.
6. **Supervisor** (if defined) — Configures supervisor processes.
7. **Uvicorn** (if defined) — Configures uvicorn supervisor services.
8. **uWSGI** (if defined) — Configures uWSGI.
9. **Celery** (if defined) — Configures Celery workers.
10. **Health Check** — Verifies uvicorn instances are responding (opt-in).

## Health Checks

The final task (`Application | Wait for uvicorn health endpoint`) polls the HTTP health endpoint of each uvicorn instance after all setup and deploy tasks have run. It is designed to catch silent failures where the service starts but is unable to handle requests (e.g. broken virtualenv, missing dependency, import error at runtime).

A handler (`verify uvicorn health`) also listens for the `restart uvicorn` handler and runs the same check immediately after a uvicorn restart, as an extra verification layer.

### Opt-in Interface

Health checks are disabled by default. Enable them per uvicorn instance by setting `health_check: true` on the instance:

```yaml
uvicorn:
  - name: "myapp"
    host: localhost
    port: 25700
    health_check: true
    health_check_path: "/health"
```

### Configuration Defaults

Override any of these on the instance, or set role-level defaults in `roles/application/defaults/main.yml`:

| Variable | Default | Description |
|----------|---------|-------------|
| `health_check` | `false` | Opt-in flag |
| `health_check_path` | `/health` | URL path to poll |
| `health_check_expected_status` | `200` | Acceptable HTTP status code |
| `health_check_timeout` | `10` | Per-request timeout (seconds) |
| `health_check_retries` | `10` | Number of retry attempts |
| `health_check_delay` | `3` | Seconds between retries |
| `health_check_validate_certs` | `false` | Validate HTTPS certificates |

Total maximum wait: `retries × delay = 10 × 3 = 30s` by default.

### How It Works

- Uses `ansible.builtin.uri` to send an HTTP GET to `http://{{ item.host }}:{{ item.port }}{{ health_check_path }}`.
- Retries up to `retries` times with `delay` seconds between each attempt.
- Fails the Ansible run if the endpoint does not return `status_code` within the timeout.
- Skips instances where `item.health_check` is not `true` or `item.remove` is `true`.

### Requirements for the Application

The application must expose an HTTP endpoint at `health_check_path` that returns `health_check_expected_status` (default `200`) when the service is ready. For example, in a FastAPI app:

```python
@app.get("/health")
async def health():
    return {"status": "ok"}
```

No application-side code changes are required beyond adding this endpoint.

## Key Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `project_name` | Yes | Used to name supervisor services, e.g. `{{ project_name }}-{{ item.name }}-uvicorn` |
| `project_user` | Yes | OS user who owns the application files and runs the services |
| `virtualenv` | Yes | Path to the Python virtualenv |
| `repo` | Yes | Dict with `url`, `version`, `force` for the git repository |
| `package_manager` | No | One of `pipenv`, `pip`, or `uv` (default: `pipenv`) |
| `django` | No | Django configuration dict |
| `fastapi` | No | FastAPI configuration dict |
| `uvicorn` | No | List of uvicorn instance dicts |
| `uwsgi` | No | uWSGI configuration dict |
| `celery` | No | Celery configuration dict |
| `supervisor` | No | Supervisor configuration dict |

## Usage Example

```bash
# Full deploy (sets up + deploys + starts services)
./play auxo production coord.deploy

# Setup only (git clone, virtualenv, no service restart)
./play auxo production coord.setup

# Start services after a manual restart
./play auxo production coord.deploy --tags start,deploy
```

## Files

- `tasks/main.yml` — Main task list
- `handlers/main.yml` — Handler list (restart uvicorn, restart celery, etc.)
- `defaults/main.yml` — Default variables
- `vars/` — Non-default variable files (not used in all environments)

## Related Documentation

- [UV_PACKAGE_MANAGER.md](./UV_PACKAGE_MANAGER.md) — uv package manager role
- [auxo/production/inventory](../metaphase/auxo/production/inventory) — Real-world example with health checks enabled
