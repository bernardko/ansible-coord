# Ansible Collection for deploying Django and FastAPI projects

A collection of roles and playbooks used for deploying Django and FastAPI projects.

## Features

- Configure and deploy Django or FastAPI projects using pipenv, poetry, uv, supervisor and uvicorn.
- Use any python version with pyenv, or let uv manage Python versions.
- Easy configuration of Celery workers.
- PostgreSQL, user and database configuration.
- RabbitMQ vhost and user configuration.
- Configure memcache instances
- Custom nginx configuration

## Installation

Installing from this repository:
```bash
ansible-galaxy install git+https://github.com/bernardko/ansible-coord.git,main
```

Using a `requirements.yml` file:
```yaml
---
collections:
  - name: https://github.com/bernardko/ansible-coord.git
    type: git
    version: main
```

Please refer to [ansible-coord-starter](https://github.com/bernardko/ansible-coord-starter) repository for full demonstration on how to use this Ansible collection.
