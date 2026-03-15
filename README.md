# ansible-monitoring-stack

![Ansible](https://img.shields.io/badge/Ansible-%3E%3D2.14-red?logo=ansible)
![Docker](https://img.shields.io/badge/Docker-Required-blue?logo=docker)
![Prometheus](https://img.shields.io/badge/Prometheus-Monitoring-orange?logo=prometheus)
![Grafana](https://img.shields.io/badge/Grafana-Dashboard-yellow?logo=grafana)
![Python](https://img.shields.io/badge/Python-3.x-blue?logo=python)

## Description

Automated deployment of an industry-standard monitoring stack using **Ansible**, targeting **Docker containers as managed hosts** (no SSH, no VMs, no cloud).

This project deploys the **Prometheus + Node Exporter + Alertmanager + Grafana** stack -the de facto standard in the modern DevOps/Kubernetes ecosystem. It appears in the vast majority of DevOps job postings and is the monitoring foundation behind most production Kubernetes clusters.

Everything runs locally via Docker, making it easy to demo, test, and explain in interviews.

## Architecture

```mermaid
graph LR
    Ansible -- deploy --> Network["Docker Network: monitoring"]

    subgraph Network["Docker Network: monitoring"]
        NE[node-exporter\n:9100]
        AM[alertmanager\n:9093]
        GF[grafana\n:3000]
        PR[prometheus\n:9090]

        PR -- scrapes --> NE
        PR -- sends alerts --> AM
        PR -- datasource --> GF
    end

    style Ansible fill:#e74c3c,color:#fff,stroke:none
    style PR fill:#e6522c,color:#fff,stroke:none
    style NE fill:#e6522c,color:#fff,stroke:none
    style AM fill:#e6522c,color:#fff,stroke:none
    style GF fill:#f9a825,color:#fff,stroke:none
    style Network fill:#f0f4f8,stroke:#3498db,stroke-width:2px
```

**How it works:**
- **Ansible** provisions and configures all containers via the Docker socket (`community.docker` collection) -no SSH needed.
- **Prometheus** scrapes metrics from Node Exporter, evaluates alert rules, and forwards firing alerts to Alertmanager.
- **Grafana** queries Prometheus as a datasource for visualization dashboards.
- All containers communicate over a dedicated Docker network (`monitoring`).

## Prerequisites

- **Docker** installed and running
- **Ansible ≥ 2.14** with Python 3.x
- **community.docker** Ansible collection

## Installing Ansible

### Option A: Local install (pip)

```bash
pip install ansible
ansible-galaxy collection install community.docker
```

### Option B: Without installing (Docker)

You can run Ansible directly from a Docker container without installing anything on your machine.

```bash
docker run --rm -it \
  -v "$(pwd)":/ansible \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -w /ansible \
  cytopia/ansible:latest \
  ansible-playbook playbooks/deploy.yml --vault-password-file .vault_pass
```

To make this easier, add an alias to your shell profile (`~/.bashrc` or `~/.zshrc`):

```bash
alias ansible-playbook='docker run --rm -it \
  -v "$(pwd)":/ansible \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -w /ansible \
  cytopia/ansible:latest \
  ansible-playbook'
```

Then use it normally:

```bash
ansible-playbook playbooks/deploy.yml --vault-password-file .vault_pass
```

## Setup - Step by Step

### 1. Install the required Ansible collection

```bash
ansible-galaxy collection install community.docker
```

### 2. Create the Vault password file

```bash
echo 'your-secure-vault-password' > .vault_pass
chmod 600 .vault_pass
```

### 3. Encrypt the secrets file

```bash
ansible-vault encrypt inventory/group_vars/all.vault.yml --vault-password-file .vault_pass
```

### 4. Deploy the monitoring stack

```bash
ansible-playbook playbooks/deploy.yml --vault-password-file .vault_pass
```

### 5. Verify all services are healthy

```bash
ansible-playbook playbooks/healthcheck.yml --vault-password-file .vault_pass
```

### 6. Access the services

| Service        | URL                          |
|----------------|------------------------------|
| Prometheus     | http://localhost:9090         |
| Grafana        | http://localhost:3000         |
| Alertmanager   | http://localhost:9093         |
| Node Exporter  | http://localhost:9100/metrics |

Default Grafana credentials: `admin` / *(password from vault)*

### 7. Tear down the stack

```bash
ansible-playbook playbooks/teardown.yml --vault-password-file .vault_pass
```

## Ansible Concepts Demonstrated

| Concept | Where in the project | Why it matters |
|---|---|---|
| Roles | `roles/prometheus/`, `roles/grafana/`... | Reusability, separation of concerns |
| Inventory | `inventory/hosts.ini` | Defines managed hosts (here: Docker containers) |
| group_vars | `inventory/group_vars/all.yml` | Shared variables across all roles |
| Ansible Vault | `all.vault.yml` | Secret encryption -never commit plaintext secrets |
| Templates Jinja2 | `prometheus.yml.j2`, `datasource.yml.j2`... | Dynamic configs injected from variables |
| Handlers | `roles/*/handlers/main.yml` | Conditional reload -avoids unnecessary restarts |
| Idempotence | All playbooks | Run N times with no side effects |
| Module `uri` | `healthcheck.yml` | HTTP verification without external dependencies |

## Screenshots

![Prometheus Targets](docs/prometheus_targets.png)

![Grafana Dashboard](docs/grafana_dashboard.png)

![Alertmanager UI](docs/alertmanager.png)

![Healthcheck Output](docs/healthcheck_output.png)

## Project Structure

```
ansible-monitoring-stack/
├── inventory/
│   ├── hosts.ini
│   └── group_vars/
│       ├── all.yml
│       └── all.vault.yml
├── roles/
│   ├── prometheus/
│   ├── alertmanager/
│   ├── node-exporter/
│   └── grafana/
├── playbooks/
│   ├── deploy.yml
│   ├── teardown.yml
│   └── healthcheck.yml
├── .vault_pass          # (in .gitignore)
├── .gitignore
├── ansible.cfg
└── README.md
```

## License

MIT
