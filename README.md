# Ansible Collection for deploying Django projects

A collection of roles and playbooks used for deploying Django projects.

## Features

- Configure and deploy Django projects using pipenv, supervisor and uvicorn.
- Use any python version with pyenv.
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