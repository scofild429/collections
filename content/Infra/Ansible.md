---
title: "Ansible"
---

# Ansible

## Configuration override order for all parameters (lowest to highest priority)

### order

Config file search order:

- *Priority lowest*: /etc/ansible/ansible.cfg — default global configuration file.
- *Priority low*: ~/.ansible.cfg — per-user configuration in the home directory.
- *Priority high*: ansible.cfg in the current working directory (project folder) overrides the above.
- *Priority highest*: `$ANSIBLE_CONFIG` — custom config file path. If set, Ansible ignores the default search paths and uses only this file.
  - Example: `ANSIBLE_CONFIG=/work/newname.cfg ansible-playbook ...`

Parameter value override order:

- Values from the selected config file are used.
- Environment variables for individual settings (e.g., `ANSIBLE_HOST_KEY_CHECKING`) override config file values.

``` bash
ansible-config list
ansible-config view
ansible-config dump
```

### Ansible background

- default use SFTP, but can be force to SCP
- SCP need SSH connection do other stuffs, it is only for file transfer
- synchronize module use rsync for large file transfer

### Modules vs. Roles vs. Collections

Module  
A single script or tool that executes one specific action (e.g., `ansible.builtin.copy`).

Role  
A self-contained, reusable directory structure bundling tasks, variables, templates, and files.

Legacy Standard  
Standalone roles uploaded to Ansible Galaxy, formatted as `username.role_name`.

- Roles can be called in two ways:
  - Play-Level (Classic) Called in the `roles:` section. You can pass overriding variables directly beneath the role name.

    ``` yaml
    - hosts: webservers
      roles:
        - role: namespace.collection.webserver
          webserver_port: 8080
    ```

  - Task-Level (Dynamic) Triggered inside the `tasks:` list using a module, allowing execution amidst other normal tasks.

    ``` yaml
    tasks:
      - name: Call a role mid-play
        ansible.builtin.include_role:
          name: namespace.collection.webserver
    ```

Collection  
The modern "shipping container" used to distribute Ansible content. A collection can contain multiple roles, modules, and plugins.

Modern Standard  
Collections. They use a Fully Qualified Collection Name (FQCN) formatted as `namespace.collection.item`.

- **Namespace:** The organization/creator (e.g., `community`, `amazon`).
- **Collection:** The specific tool bundle (e.g., `docker`, `aws`).
- **Item:** The role or module being called.
- **Example:** `community.docker.docker_container` (Module) or `middleware_automation.jcliff.nginx` (Role).

- Strict Execution Order Ansible guarantees a specific execution order at the play level. All roles completely finish before standard tasks begin.
  1.  `pre_tasks:` (Setup/prerequisites)
  2.  `roles:` (All play-level roles execute fully)
  3.  `tasks:` (Standard modules execute here)
  4.  `post_tasks:` (Verification/cleanup)

## Craft import — Ansible — 2026-09-27

<!-- Org properties: {"craft_id": "7089CF11-90D2-4774-81DB-147B28EFE7D4", "imported": "2026-09-27"} -->

Source: [Ansible in Craft](craftdocs://open?spaceId=f0e27734-d8b8-47ce-be9d-9b35def0cb70&blockId=7089CF11-90D2-4774-81DB-147B28EFE7D4)

### Architecture

------------------------------------------------------------------------

#### Collection

- the default release and install package, modern "shipping container"

- can contains multiple roles, modules, plugins and others

- use Fully Qualified Collection Name

  Namespace.Collection.Item

- execution order

  pre tasks → roles → tasks → post tasks

``` text
ansible-galaxy collection install namespace.collection
```

------------------------------------------------------------------------

#### Role

- A self-contained, reusable directory structure bundling tasks, variables, templates, and files
- can be reused with release and install
  - Legacy standard, upload to ansible galaxy with username.rolename

``` text
ansible-galaxy install username.rolename
```

- can be reused in two ways:
  - called in the **role** section

``` text
- hosts: webservers
  roles:
    - role: namespace.collection.webserver
      webserver_port: 8080
```

- can be triggered inside the **tasks** list as a module

``` text
tasks:
  - name: Call a role mid-play               
    ansible.builtin.include_role:
      name: namespace.collection.webserver
```

#### Configuration priority

- Lowest: /etc/ansible/ansible.cfg. default global configuration file.
- low: ~/.ansible.cfg. per-user configuration in the home directory.
- High: ansible.cfg current working directory , project folder.
- Higher: \$ANSIBLE<sub>CONFIG</sub>. custom config file path.
  - If set, Ansible ignores the default search paths and uses only this file.
  - Example: ANSIBLE<sub>CONFIG</sub>=/work/newname.cfg ansible-playbook.
- Highest: Environment variables
  - override the values from the config file
  - can be set with ANSIBLE<sub>HOST</sub>\_KEY<sub>CHECKING</sub>
  - can passed by the terminal command
  - can be inherited by system environment varible

``` text
ansible-config list
ansible-config view
ansible-config dump
```

------------------------------------------------------------------------

#### Network connection

- default use SFTP, but can be force to SCP
- SCP need SSH connection do other stuffs, it is only for file transfer
- synchronize module use rsync for large file transfer
