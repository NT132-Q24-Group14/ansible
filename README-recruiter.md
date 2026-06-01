# Ansible CI/CD Deployment Automation

This repository demonstrates an Ansible-based deployment automation setup for a containerized application. It is designed to be executed from a Jenkins CI/CD pipeline and focuses on practical deployment operations such as release deployment, rollback, rollforward, runtime configuration, and traffic switching through Nginx.

The project represents the infrastructure automation layer of a DevOps workflow. Jenkins handles pipeline orchestration and passes runtime values, while Ansible applies the required changes on the target application server.

## Project Highlights

- Automated deployment of a Dockerized application to a remote Linux server.
- Jenkins-driven execution model for CI/CD integration.
- Rollback and rollforward playbooks for controlled release recovery.
- Nginx upstream switching to route traffic between active application versions.
- Secret management through Ansible Vault.
- Environment-specific inventory configuration for remote server access.
- Clear separation between orchestration, deployment logic, runtime configuration, and encrypted secrets.

## What This Project Shows

This project is not only a set of Ansible scripts. It shows how deployment automation can be structured as a maintainable part of a CI/CD system.

The repository demonstrates the ability to:

- automate application deployment using Ansible playbooks;
- integrate Ansible execution with Jenkins pipeline variables;
- manage Docker containers on a remote server;
- support rollback and rollforward operations with explicit image tags;
- update Nginx routing without manually editing server configuration;
- handle sensitive database values through encrypted Ansible Vault files;
- keep infrastructure configuration version-controlled and repeatable.

## Technology Stack

| Area | Technology |
| --- | --- |
| CI/CD orchestration | Jenkins |
| Deployment automation | Ansible |
| Runtime platform | Docker |
| Traffic routing | Nginx |
| Secrets management | Ansible Vault |
| Target environment | Remote Linux application server |

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

## Deployment Model

Jenkins is expected to call the Ansible playbooks during the pipeline. Most runtime values are provided by Jenkins as environment variables or Ansible extra variables.

The deployment responsibility is split as follows:

| Component | Responsibility |
| --- | --- |
| Jenkins | Selects the deployment action and passes runtime values |
| Ansible | Executes deployment, rollback, and rollforward operations |
| Docker | Runs the application container |
| Nginx | Routes traffic to the active container |
| Ansible Vault | Protects database configuration values |

This separation makes the deployment process easier to understand, automate, and recover when a release needs to be reverted.

## Main Workflows

### Deploy

The deploy workflow is intended for the main branch. Jenkins provides the branch context through `BRANCH_NAME`, and the playbook only starts the application when the branch is `main`.

The playbook:

1. Pulls the latest stable Docker image.
2. Starts or recreates the main application container.
3. Injects database configuration from Vault-backed variables.
4. Exposes the application on host port `8080`.

Manual equivalent:

```bash
BRANCH_NAME=main ansible-playbook playbooks/deploy-app.yml --ask-vault-pass
```

### Rollback

The rollback workflow allows Jenkins or an operator to move traffic to a previous Docker image tag.

The playbook:

1. Requires an `image_tag` value.
2. Pulls the selected rollback image.
3. Starts the rollback container on host port `8081`.
4. Updates the Nginx upstream to `localhost:8081`.
5. Reloads Nginx.
6. Removes the previous main container.

Manual equivalent:

```bash
ansible-playbook playbooks/rollback-app.yml -e "image_tag=<tag>" --ask-vault-pass
```

### Rollforward

The rollforward workflow moves the application back to the main container path after a recovery or a new validated release.

The playbook:

1. Requires an `image_tag` value.
2. Pulls the selected image.
3. Starts the main application container on host port `8080`.
4. Updates the Nginx upstream to `localhost:8080`.
5. Reloads Nginx.
6. Removes the rollback container.

Manual equivalent:

```bash
ansible-playbook playbooks/rollforward-app.yml -e "image_tag=<tag>" --ask-vault-pass
```

## Runtime Configuration

The application image is hosted under:

```text
mothmon14682/pms
```

The main deployment uses:

| Setting | Value |
| --- | --- |
| Main container | `pms` |
| Rollback container | `pms_rollback` |
| Main port | `8080` |
| Rollback port | `8081` |
| Nginx site config | `/etc/nginx/sites-available/pms` |

Database values are passed into the container as environment variables:

| Container variable | Source variable |
| --- | --- |
| `pms_db_host` | `db_host` |
| `pms_db_name` | `db_name` |
| `pms_db_username` | `db_username` |
| `pms_db_password` | `db_password` |

The public variable file references encrypted values from Ansible Vault, keeping sensitive configuration out of plaintext playbooks.

## Operational Value

From a DevOps perspective, this repository focuses on release reliability and recovery:

- Deployments are repeatable because the steps are captured in version-controlled playbooks.
- Rollback is explicit because the target image tag must be provided.
- Traffic switching is automated through Nginx configuration updates.
- Secrets are separated from deployment logic.
- Jenkins can standardize execution across environments by passing variables into Ansible.

This makes the repository suitable as a foundation for a small production-style deployment pipeline or a DevOps course project that demonstrates practical release operations.

## Validation

A successful run should be verified by checking:

- Ansible completes without failed tasks.
- The expected Docker container is running on the target server.
- The active host port is available.
- Nginx reloads successfully.
- The application is reachable through the configured server endpoint.
- For rollback or rollforward, the Nginx upstream points to the expected container port.

## Notes for Reviewers

This repository focuses on deployment automation rather than application source code. The build, test, and Docker image publishing stages are expected to be handled before these playbooks run, typically by Jenkins.

The inventory values such as host address, SSH username, and SSH key path are environment-specific. In a real CI/CD environment, these values should be managed together with Jenkins credentials and deployment environment configuration.
