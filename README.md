# Ansible FreeIPA and NFS Home Directory Automation

Ansible automation for deploying a RHEL-based FreeIPA identity environment with NFS/autofs-based roaming home directories.

## Overview

This project builds a Linux identity management lab using Ansible, FreeIPA, NFS, and autofs. The automation deploys a FreeIPA server, enrolls client systems, provisions FreeIPA users, configures client-based NFS home exports, and validates that users can access the same roaming home directory from multiple client machines.

The goal is to demonstrate repeatable Linux infrastructure automation with a focus on identity management, centralized authentication, NFS home directory access, autofs path resolution, and role-based Ansible design.

## Architecture

```text
                         +-----------------------------+
                         |       Ansible Control       |
                         |   ansible01.example.test    |
                         +--------------+--------------+
                                        |
                                        | Ansible SSH
                                        |
              +-------------------------+--------------------------+
              |                                                    |
              v                                                    v
+-----------------------------+              +----------------------------------+
| FreeIPA Server              |              | FreeIPA Clients                  |
| ipa01.example.test          |              | client01.example.test            |
|                             |              | client02.example.test            |
| - FreeIPA server            |              | client03.example.test            |
| - Kerberos/KDC              |              |                                  |
| - LDAP identity backend     |              | - FreeIPA client enrollment      |
| - DNS/identity services     |              | - NFS /home exports              |
+-----------------------------+              | - autofs /net path resolution    |
                                             | - roaming home directory access  |
                                             +----------------------------------+
```

## Technologies Used

* Red Hat Enterprise Linux 9
* Ansible
* FreeIPA
* Kerberos
* LDAP
* NFS
* autofs
* firewalld
* SELinux
* Jinja2 templates
* Ansible Vault
* Git/GitHub

## Repository Structure

```text
.
├── ansible.cfg
├── docs/
│   └── roaming-home-directory-validation.md
├── inventory/
│   ├── group_vars/
│   │   ├── all.yml
│   │   └── ipaclients.yml
│   └── hosts.ini
├── playbooks/
│   ├── create-freeipa-users.yml
│   ├── enroll-freeipa-clients.yml
│   ├── install-freeipa-server.yml
│   └── site.yml
├── roles/
│   └── nfs_server/
│       ├── defaults/
│       ├── handlers/
│       ├── tasks/
│       ├── templates/
│       └── vars/
├── vars/
│   ├── users.yml
│   ├── users_vault.yml
│   └── vault.yml
├── requirements.yml
├── LICENSE
└── README.md
```

## Inventory Design

The lab uses one FreeIPA server and three FreeIPA client systems.

```ini
[ipaservers]
ipa01.example.test

[ipaclients]
client01.example.test
client02.example.test
client03.example.test

[lab:children]
ipaservers
ipaclients
```

The FreeIPA clients are configured with NFS and autofs to support roaming home directory access through `/net` paths.

## What This Automation Builds

The automation performs the following:

* Deploys a FreeIPA server
* Enrolls RHEL client systems into the FreeIPA domain
* Provisions FreeIPA users with Vault-backed temporary passwords
* Configures FreeIPA user `homedir` attributes
* Creates physical home directories on assigned client systems
* Configures client-based NFS `/home` exports
* Configures autofs with the `/net -hosts` map
* Sets the SELinux boolean required for NFS-backed home directories
* Opens required firewall services for NFS traffic
* Configures NFSv4 ID mapping with the FreeIPA domain
* Validates cross-client roaming home directory access

## User Home Directory Design

Each FreeIPA user is assigned a roaming home directory path that resolves through autofs.

| User    | Home Host             | FreeIPA Home Directory                  |
| ------- | --------------------- | --------------------------------------- |
| nwright | client01.example.test | /net/client01.example.test/home/nwright |
| kellis  | client03.example.test | /net/client03.example.test/home/kellis  |

The physical directories are created on the assigned client systems:

```text
/home/nwright -> client01.example.test
/home/kellis  -> client03.example.test
```

When the same user logs in from another enrolled client, autofs resolves the `/net/HOSTNAME/home/USERNAME` path back to the assigned home host.

## Ansible Design

This project uses wrapper playbooks to keep the automation modular and easier to maintain.

| Playbook                     | Purpose                                    |
| ---------------------------- | ------------------------------------------ |
| `install-freeipa-server.yml` | Deploys the FreeIPA server                 |
| `enroll-freeipa-clients.yml` | Enrolls client systems into FreeIPA        |
| `create-freeipa-users.yml`   | Creates FreeIPA users and home directories |
| `site.yml`                   | Runs the full end-to-end workflow          |

The `nfs_server` role manages NFS and autofs configuration through a reusable Ansible role structure.

Key role functions include:

* Installing `nfs-utils` and `autofs`
* Enabling `use_nfs_home_dirs`
* Starting and enabling required services
* Rendering `/etc/exports` from a Jinja2 template
* Rendering `/etc/auto.master` from a Jinja2 template
* Opening NFS-related firewall services
* Updating `/etc/idmapd.conf`
* Running handlers only when related configuration changes

## Security Design

Sensitive values are stored with Ansible Vault instead of being hardcoded in playbooks.

Vault-backed values include:

* FreeIPA admin password
* Directory Manager password
* Temporary FreeIPA user password

The FreeIPA user creation task uses `no_log: true` to avoid exposing passwords in Ansible output.

Non-sensitive identity data, such as usernames and display names, is stored separately from encrypted password values.

This repository uses Ansible Vault for sensitive values such as FreeIPA administrator credentials and temporary user passwords. Vault files should be replaced with environment-specific encrypted values before reuse outside this lab.

## Ansible Collection Dependencies

Required Ansible collections are defined in `requirements.yml`.

Install the collections locally with:

```bash
ansible-galaxy collection install -r requirements.yml -p collections/
```

The `collections/` directory is used as a local dependency install path and is not intended to be committed to the repository.

## Prerequisites

To run this project, the environment should have:

* RHEL 9 systems with active subscriptions
* SSH access from the Ansible control node to managed nodes
* Passwordless sudo or appropriate privilege escalation configured
* Python available on managed nodes
* Ansible installed on the control node
* Required Ansible collections installed from `requirements.yml`
* Required inventory and Vault variables configured
* A local vault password file configured for the environment

The commands below assume a local vault password file exists at `~/ansible-secrets/freeipa-nfs-vault-pass.txt`. Adjust the path as needed for your environment.

## Running the Automation

Run commands from the repository root.

### Syntax check

```bash
ansible-playbook playbooks/site.yml --vault-password-file ~/ansible-secrets/freeipa-nfs-vault-pass.txt --syntax-check
```

### Run the full workflow

```bash
ansible-playbook playbooks/site.yml --vault-password-file ~/ansible-secrets/freeipa-nfs-vault-pass.txt
```

### Run only the NFS/autofs configuration

```bash
ansible-playbook playbooks/site.yml --tags nfs --limit ipaclients --vault-password-file ~/ansible-secrets/freeipa-nfs-vault-pass.txt
```

## Validation

Roaming home directory validation confirmed:

* FreeIPA users can authenticate from multiple client systems
* FreeIPA `homedir` attributes point to the correct `/net` paths
* Physical home directories exist on the assigned home hosts
* autofs resolves `/net` paths from all FreeIPA clients
* Test files persist across client logins for the same user
* NFS exports, autofs, SELinux, and file ownership work together successfully

Detailed validation results are documented in [roaming-home-directory-validation.md](docs/roaming-home-directory-validation.md).

## Skills Demonstrated

This project demonstrates:

* Linux identity management with FreeIPA
* RHEL system administration
* Ansible playbook and role development
* Ansible Vault usage
* FreeIPA user and client automation
* NFS and autofs configuration
* SELinux boolean management
* firewalld service management
* Jinja2 templating
* Handler-based service reloads
* Cross-client validation and documentation
* Git-based infrastructure documentation

## Result

This project provides a repeatable Ansible workflow for deploying a FreeIPA-based Linux identity lab with NFS/autofs-based roaming home directories. The final workflow ties together identity management, client enrollment, home directory provisioning, NFS exports, autofs resolution, and end-to-end validation.

