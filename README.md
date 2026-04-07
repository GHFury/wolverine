# Wolverine

Server provisioning and hardening using Ansible roles with Docker-based local testing.

## Overview

This project demonstrates automated server configuration using Ansible. Five reusable roles take a fresh Ubuntu server from zero to production-ready, covering base packages, user management, SSH hardening, Docker installation, and Datadog agent setup. Docker containers simulate target servers locally, allowing full playbook development and testing without cloud infrastructure.

## Stack

| Layer | Technology |
|---|---|
| Configuration Management | Ansible |
| Target OS | Ubuntu 22.04 |
| Local Testing | Docker / Docker Compose |
| Monitoring Agent | Datadog |
| Container Runtime | Docker CE |
| Security | UFW / fail2ban / SSH hardening |

## Roles

### common

Installs base packages, sets the timezone, and updates the apt cache. Packages include curl, wget, vim, git, htop, and networking tools.

### users

Creates an ops user with sudo privileges and deploys an SSH public key for passwordless authentication. This role runs before security hardening to ensure key-based access is in place before password login is disabled.

### security

Hardens SSH by disabling root login, disabling password authentication, and limiting max auth tries to 3. Installs fail2ban for brute force protection. Configures UFW firewall to deny all incoming traffic except SSH. UFW tasks detect Docker containers and skip automatically since containers share the host kernel's network stack.

### docker

Adds the official Docker GPG key and repository, installs Docker CE, and configures the daemon with log rotation and address pool settings. Adds the ops user to the docker group for rootless container management.

### datadog

Installs the Datadog agent from the official APT repository. Configures the agent using Jinja2 templates with variables for API key, site, APM, log collection, and process monitoring. All settings are customizable through role defaults.

## Pipeline

Roles execute in a specific order that matters:

1. common — base packages and timezone
2. users — ops user and SSH key (must exist before SSH lockdown)
3. security — SSH hardening and firewall
4. docker — container runtime
5. datadog — monitoring agent

## Getting started

```bash
git clone git@github.com:GHFury/Wolverine.git
cd Wolverine
```

Generate an SSH key for Ansible to deploy to target servers:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/wolverine_key -N ""
```

Install sshpass for initial password-based connection:

```bash
sudo apt install sshpass -y
```

Spin up the test servers and run the playbook:

```bash
docker compose up -d --build
export ANSIBLE_CONFIG=$(pwd)/ansible.cfg
ansible-playbook site.yml
```

Verify connectivity before running the full playbook:

```bash
ansible all -m ping
```

## Project structure

```
ansible.cfg              Ansible configuration
docker-compose.yml       Test server definitions
docker/Dockerfile        Ubuntu SSH target image
inventory.ini            Server inventory
site.yml                 Main playbook
roles/
  common/tasks/          Base packages, timezone
  users/tasks/           Ops user, SSH keys, sudo
  security/tasks/        SSH hardening, fail2ban, UFW
  security/handlers/     SSH reload handler
  docker/tasks/          Docker CE installation
  datadog/defaults/      Configurable variables
  datadog/tasks/         Agent installation
  datadog/templates/     Jinja2 config templates
```

## Testing environment

The Docker Compose file spins up two Ubuntu 22.04 containers with SSH enabled, simulating fresh servers. Ansible connects to them over SSH on ports 2221 and 2222 just as it would against real infrastructure.

Some tasks behave differently in containers than on real servers. UFW and systemd services are automatically detected and skipped since Docker containers share the host kernel. The SSH handler uses SIGHUP to reload configuration instead of restarting the service, since containers run sshd as PID 1.

## Configuration

The Datadog role accepts the following variables through `roles/datadog/defaults/main.yml`:

```yaml
datadog_api_key: "YOUR_API_KEY_HERE"
datadog_site: "datadoghq.com"
datadog_apm_enabled: true
datadog_logs_enabled: true
datadog_process_enabled: true
```

Set the API key via environment variable:

```bash
DD_API_KEY=your_key_here ansible-playbook site.yml
```

## Lessons learned

Ansible ignores `ansible.cfg` in world-writable directories for security. WSL mounts Windows paths as world-writable, so the config file must be referenced explicitly with the `ANSIBLE_CONFIG` environment variable.

Password-based SSH access requires sshpass to be installed on the control machine. This is only needed for initial provisioning before SSH keys are deployed.

Docker containers lack full kernel access for iptables and systemd. Roles that manage firewalls or system services need conditionals to detect containers and skip gracefully. Checking for `/.dockerenv` is a reliable detection method.

The SSH handler must reload rather than restart sshd in containers. Since sshd runs as PID 1, restarting it kills the container. Sending SIGHUP reloads the configuration without dropping the process.

Role execution order matters. The users role must run before security hardening, otherwise disabling root login and password authentication locks out the only available access method.

## Development notes

To reset the test environment and start fresh:

```bash
docker compose down
docker compose up -d --build
ansible-playbook site.yml
```

To add a new role, create the directory structure under `roles/` and add it to the `site.yml` playbook. Roles execute in the order listed.
