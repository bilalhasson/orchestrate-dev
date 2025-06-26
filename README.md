# Ansible Playbook for macOS Setup

# Problem

Setting up my laptop for work took way too long, getting the right tools,
logging into the correct systems, cloning the right repos, etc.

# Solution

This repository contains an Ansible playbook to set up a macOS machine with various tools and configurations.
It is designed to automate the installation of essential software and configurations for development environments.

What I do when you execute the main playbook:

1. Install:
   - VSCode
   - Homebrew
   - dependencies like curl, unzip and python `packaging` to allow for AWS install
   - AWS CLI
2. Clone repositories defined in the `group_clone_repos` variable.
3. Fetch environment variables from AWS Secrets Manager and create `.env` files defined in the `group_clone_repos` variable.
4. Install Python packages using `pipenv` or `npm` as specified in the `group_clone_repos` variable.
5. Setup the local development environment, `npm i` and `pipenv install`.
6. Install Postgres version defined in the `group_postgres_version` variable.
7. Install docker
8. Build and run Docker containers for the cloned repositories, using the commands specified in the `group_clone_repos` variable.
9. Make fixtures for the repositories, if specified in the `group_clone_repos` variable.
10. Install postman

# Table of Contents

- [Prerequisites](#prerequisites)
- [Run Ansible Playbook](#run-ansible-playbook-to-set-up-a-macos-machine)
- [Install Python and Ansible on macOS](#install-python-and-ansible-on-macos)
- [Ansible Structure](#ansible-structure)
- [Roles](#roles)
- [Variables](#variables)

# Prerequisites

1. Python
2. Create venv for Ansible
   - `python -m venv ansible`
   - `source ansible/bin/activate`
3. Ansible
   - `pip install pipx`
   - `pipx install ansible`
4. SSH key for GitHub
   - Create SSH key and add it to your GitHub account
   - Add SSH key to your macOS keychain
     ```bash
     ssh-add ~/.ssh/<your-ssh-key>
     ```

# Run ansible playbook to set up a macOS machine

Run the below command, and then enter your password when prompted.

```bash
ansible-playbook -i inventory playbooks/macos/main.yml --ask-become-pass
```

# Install Python and Ansible on macOS

1. Create a Python virtual environment:
   ```bash
   python3 -m venv ansible
   source ansible/bin/activate
   ```
2. Upgrade pip (recommended):
   ```bash
   pip install --upgrade pip
   pip install pipx
   ```
3. Install Ansible inside the virtual environment:
   ```bash
   pipx install ansible
   ```

# Ansible Structure

      ansible/
      group_vars/
         all/
            env_vars.yml
            env_vars.local.yml
      host_vars/
         localhost/
            env_vars.yml
      inventory/
         hosts
      playbooks/
         macos/
            main.yml
         roles/
            aws_login/
               main.yml
         playbooks.yml

# Roles

- `aws_login`: Configures AWS CLI with login credentials.

# Variables

Create variables in the `group_vars/all/env_vars.yml`.

```yaml
python_interpreter: ".../bin/python"

# Directory for all the repos.
group_develop_dir: ".../repos/location"

# If SSO is required, set to true.
group_aws_sso_login_required: true

# AWS SSO
group_aws_details:
  region: eu-west-2
  output: json
  sso_start_url: https://d-9c6714caab.awsapps.com/start
  sso_region: eu-west-2
  sso_account_id: <sso_account_id>
  sso_role_name: <sso_role_name>

  ecr_login_url: <id>.dkr.ecr.<region>.amazonaws.com

# Default is `/tmp`, change if you want to use a different directory for temporary files.
override_executable_temp_dir: /tmp

# This is used to prefix the repo name when cloning repos.
group_repo_organization: <org or user>

group_clone_repos:
  backend-repo-name:
    service: "backend"
    install_pipenv: true # If pipenv is used, and you want to install set to true.
    pipenv_subdir: <sub_dir> # Specify the subdirectory of Pipfile if applicable.
    env_secret: # Environment secret for the repo. Uses aws secretsmanager.
      id: secret-id
      file: "{{ [group_develop_dir, '.../.env'] | path_join }}"
    load_fixtures: true
    docker:
      build_command: "make build"
      up_command: "make up"
      check_ports_before_fixtures:
        - port: 8001 # web
        - port: 5433 # db

  frontend-repo-name:
    service: "frontend"
    install_method: "npm"
    node_version: "20.18"
    env_secret:
      id: secret-id
      file: "{{ [group_develop_dir, 'frontend-repo-name', '.env.local'] | path_join }}"

# Postgres
group_postgres_version: 14

# If repo node version not provided, use this as a fallback.
group_fallback_node_version: "18"
```
