# Ansible Deployment Automation

This repository contains the Ansible deployment configuration for a DevOps academic project. It acts as the deployment automation layer for the `pms` application, which is packaged as a Docker image and deployed to an application server.

In the intended CI/CD workflow, Jenkins is the primary orchestrator. Jenkins triggers the appropriate Ansible playbook for deployment, rollback, or rollforward operations, while Ansible performs the server-side actions such as pulling Docker images, managing containers, updating Nginx upstream configuration, and reloading services.

## Architecture Overview

The deployment workflow is built around the following components:

- Jenkins: triggers deployment actions from the CI/CD pipeline and passes runtime values to Ansible through environment variables or extra variables.
- Ansible: connects to the target application server and executes the deployment playbooks.
- Docker: runs the application as a containerized service.
- Nginx: routes traffic to the currently active application container.
- Ansible Vault: stores sensitive database configuration values in encrypted form.

The repository is intentionally focused on infrastructure automation. Application build, test, and image publishing steps are expected to happen before these playbooks are executed, typically inside the Jenkins pipeline.

## Repository Structure

```text
.
+-- ansible.cfg
+-- inventory/
|   +-- hosts.ini
|   +-- group_vars/
|       +-- all/
|           +-- vars.yml
|           +-- vault.yml
+-- playbooks/
    +-- deploy-app.yml
    +-- rollback-app.yml
    +-- rollforward-app.yml
```

### Main Files

- `ansible.cfg`: defines the default inventory file, enables SSH host key checking, and configures YAML output formatting.
- `inventory/hosts.ini`: defines the application server target under the `app` host group. Host address, SSH user, and private key path are environment-specific values.
- `inventory/group_vars/all/vars.yml`: exposes database variables used by the playbooks and maps them to encrypted Vault values.
- `inventory/group_vars/all/vault.yml`: stores encrypted database configuration through Ansible Vault.
- `playbooks/deploy-app.yml`: deploys the stable application image to the main container.
- `playbooks/rollback-app.yml`: rolls back to a selected Docker image tag and switches Nginx traffic to the rollback container.
- `playbooks/rollforward-app.yml`: rolls forward to a selected Docker image tag and switches Nginx traffic back to the main container.

## Configuration

The current playbooks deploy the Docker image repository `mothmon14682/pms`. The default stable deployment uses the image tag `stable`.

The application container uses the following runtime configuration:

| Item | Value |
| --- | --- |
| Main container name | `pms` |
| Rollback container name | `pms_rollback` |
| Application container port | `8080` |
| Main host port | `8080` |
| Rollback host port | `8081` |
| Nginx config path | `/etc/nginx/sites-available/pms` |

Database configuration is injected into the container as environment variables:

| Container environment variable | Ansible variable |
| --- | --- |
| `pms_db_host` | `db_host` |
| `pms_db_name` | `db_name` |
| `pms_db_username` | `db_username` |
| `pms_db_password` | `db_password` |

These Ansible variables are loaded from `inventory/group_vars/all/vars.yml`, which references encrypted values from `vault.yml`.

## Jenkins Integration

This repository is designed to be called by Jenkins as part of a CI/CD pipeline. Jenkins is responsible for deciding which playbook should run and for passing runtime values required by the deployment process.

Typical responsibilities of Jenkins include:

- selecting the deployment action, such as deploy, rollback, or rollforward;
- providing the branch context through variables such as `BRANCH_NAME`;
- passing the target Docker image tag through the `image_tag` extra variable for rollback and rollforward operations;
- providing access to SSH credentials and Ansible Vault credentials through Jenkins credentials management;
- collecting deployment logs from the Ansible command output.

The deploy playbook only starts the application container when `BRANCH_NAME` is equal to `main`. This allows Jenkins to gate production-style deployment behavior based on the pipeline branch.

Rollback and rollforward playbooks require the `image_tag` variable. This value is expected to be supplied by Jenkins as an Ansible extra variable when the pipeline triggers one of these operations.

## Usage

The following commands show how the playbooks can be executed manually. In the normal workflow, Jenkins runs equivalent commands from the pipeline.

### Deploy

```bash
BRANCH_NAME=main ansible-playbook playbooks/deploy-app.yml --ask-vault-pass
```

This playbook pulls the latest `mothmon14682/pms:stable` image and runs the main `pms` container on port `8080`.

### Rollback

```bash
ansible-playbook playbooks/rollback-app.yml -e "image_tag=<tag>" --ask-vault-pass
```

This playbook pulls `mothmon14682/pms:<tag>`, starts the `pms_rollback` container on host port `8081`, updates the Nginx upstream to `localhost:8081`, reloads Nginx, and removes the previous main container.

### Rollforward

```bash
ansible-playbook playbooks/rollforward-app.yml -e "image_tag=<tag>" --ask-vault-pass
```

This playbook pulls `mothmon14682/pms:<tag>`, starts the main `pms` container on host port `8080`, updates the Nginx upstream to `localhost:8080`, reloads Nginx, and removes the rollback container.

## Deployment Flow

### Standard Deployment

1. Jenkins triggers the deploy stage for the `main` branch.
2. Ansible pulls the latest stable Docker image.
3. Ansible starts or recreates the `pms` container.
4. Database values are injected from Ansible Vault-backed variables.
5. The application is exposed on host port `8080`.

### Rollback

1. Jenkins provides the target rollback image tag as `image_tag`.
2. Ansible validates that `image_tag` is present.
3. Ansible pulls the selected Docker image version.
4. Ansible starts the `pms_rollback` container on host port `8081`.
5. Ansible waits for port `8081` to become available.
6. Ansible updates the Nginx upstream to route traffic to `localhost:8081`.
7. Ansible reloads Nginx and removes the old `pms` container.

### Rollforward

1. Jenkins provides the target rollforward image tag as `image_tag`.
2. Ansible validates that `image_tag` is present.
3. Ansible pulls the selected Docker image version.
4. Ansible starts the `pms` container on host port `8080`.
5. Ansible waits for port `8080` to become available.
6. Ansible updates the Nginx upstream to route traffic to `localhost:8080`.
7. Ansible reloads Nginx and removes the `pms_rollback` container.

## Security Notes

Sensitive database values are not stored directly in plaintext playbooks. They are referenced through normal group variables and stored in encrypted form with Ansible Vault.

In a Jenkins-based workflow, sensitive values such as SSH private keys, Vault passwords, and other deployment credentials should be managed through Jenkins credentials management. The inventory file should be treated as environment-specific configuration and updated according to the target infrastructure.

## Validation

After running a playbook, deployment success can be validated through the following checks:

- the Ansible command finishes without failed tasks;
- the expected Docker container is running on the target application server;
- the expected host port is available, either `8080` for the main container or `8081` for the rollback container;
- Nginx reloads successfully after the upstream configuration is changed;
- the application is reachable through the configured server endpoint.

For rollback and rollforward operations, the Nginx upstream should point to the container selected by the playbook.
