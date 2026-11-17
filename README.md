# uni-devops-lab

University Student Portal (Flask + Redis) managed with Git, packaged with Docker
and deployed to a server with Ansible.

## Structure

```
app/            Flask application (app.py, requirements.txt)
ansible/        Playbook, inventory, variables, templates
Dockerfile      python:3.12-slim image, dependencies installed before code
compose.yml     portal + Redis, Redis data in named volume `redis_data`
```

## Endpoints

| Path | Description |
|------|-------------|
| `/` | Status text |
| `/health` | `{"status": "healthy"}` |
| `/visits` | Redis-backed counter (survives restarts) |

## Run locally

```bash
docker build -t student-portal:1.0 .
docker compose up -d --build
curl http://localhost:8080/health
curl http://localhost:8080/visits        # 1, 2, 3 ...
docker compose restart
curl http://localhost:8080/visits        # continues, does not reset
```

Port and image name can be overridden with `PORTAL_PORT` and `IMAGE_NAME`
(defaults: `8080`, `student-portal:1.0`).

## Deploy with Ansible

Requirements on the control node: `ansible-core`, and passwordless SSH to the
server (`ssh-copy-id user@server`). The server must be Ubuntu/Debian with sudo.

```bash
cd ansible
ansible-galaxy collection install -r requirements.yml
# edit inventory.ini: set ansible_host and ansible_user
ansible all -m ping
ansible-playbook playbook.yml     # first run: installs Docker, deploys, checks /health
ansible-playbook playbook.yml     # second run: changed=0
```

Port, image name and target directory are variables in
`ansible/group_vars/all.yml` (`portal_port`, `portal_image`, `portal_dir`);
the playbook renders them into a `.env` file for Compose on the server.
The playbook ends by calling `/health` and printing the result.

After deploy: `http://<server-ip>:8080`.

## Secrets

No passwords, tokens or private keys are stored in the repository. `.env`,
`*.log`, `*.pem`, `*.key` and `id_*` files are in `.gitignore`. SSH uses key
authentication with a key that lives only on the control node.

## Problem encountered

While validating the playbook, `ansible-playbook --syntax-check` failed with
`couldn't resolve module/action 'community.docker.docker_compose_v2'`.
`ansible-core` ships without the `community.docker` collection, so the Compose
module did not exist. Fix: declare the dependency in `ansible/requirements.yml`
and install it with `ansible-galaxy collection install -r requirements.yml`
before running the playbook.
