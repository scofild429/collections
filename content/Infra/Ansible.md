---
title: "Ansible"
---

# Ansible

## Configuration selection and precedence

### Which configuration file is loaded?

Ansible searches in this order and uses the **first file found**, rather than merging the files:

1. The path specified by `ANSIBLE_CONFIG`, if set.
2. `ansible.cfg` in the current working directory.
3. `~/.ansible.cfg`.
4. `/etc/ansible/ansible.cfg`.

Do not assume that setting `ANSIBLE_CONFIG` to a missing file guarantees use of that configuration; verify the selected file with `ansible --version`. Automatic loading from a world-writable current directory is restricted. See [configuration-file discovery](https://docs.ansible.com/ansible/latest/reference_appendices/config.html).

```bash
ANSIBLE_CONFIG=/work/ansible.cfg ansible-playbook -i inventory.ini site.yml
```

The paths above are placeholders for an existing project.

### Which setting wins?

The broad categories, from lower to higher precedence, are:

| Category | Examples |
|---|---|
| Configuration | Selected `ansible.cfg`, then a supported environment override |
| Command-line options | `-u deploy` |
| Playbook keywords | `remote_user: deploy` |
| Variables | Inventory `ansible_user`, play/task variables, extra variables |
| Direct assignment | Options passed directly to a module or plugin where applicable |

This is not a rule that every parameter supports every source. Consult the setting/plugin documentation. Within variables there is a separate precedence hierarchy; `-e` extra variables have the highest variable precedence. An environment variable is not universally the highest-priority setting.

For example, `ansible_user` from inventory can override `-u`, and `-e ansible_user=deploy` overrides other definitions of that variable. A module argument such as `path` is a directly assigned option; its Jinja expression may still resolve a variable. See [Ansible precedence rules](https://docs.ansible.com/ansible/latest/reference_appendices/general_precedence.html).

### Inspect configuration

```bash
ansible --version                   # Version and selected config file
ansible-config list                 # Settings, supported sources, defaults
ansible-config view                 # Contents of the selected config file
ansible-config dump                 # Effective configuration settings
ansible-config dump --only-changed  # Settings changed from defaults
```

`view` requires a selected configuration file. `dump` is not a report of every resolved host/task variable in a playbook.

## Modules, roles, and collections

| Component | Purpose | Example |
|---|---|---|
| Module | Implements an individual action used by a task | `ansible.builtin.copy` |
| Role | Reusable structure of tasks, defaults, variables, handlers, templates, and files | Local role `webserver` |
| Collection | Distribution unit containing modules, roles, plugins, and related content | `community.docker` |

A collection is named `namespace.collection`. An item within it normally uses a fully qualified collection name (FQCN), such as `community.docker.docker_container`. `namespace.collection.webserver` below is a naming illustration, not a claim that this role exists.

Standalone roles remain supported and may be installed as `author.role_name`; they are not invalid merely because collections are also available.

```bash
# Illustrative names: replace with actual content you intend to install.
ansible-galaxy collection install namespace.collection
ansible-galaxy role install author.role_name
```

### Role layout

```text
roles/webserver/
├── tasks/main.yml
├── defaults/main.yml
├── vars/main.yml
├── handlers/main.yml
├── templates/
├── files/
└── meta/main.yml
```

Include only the directories needed. `defaults/main.yml` supplies easily overridden defaults; `vars/main.yml` has higher precedence. These are not interchangeable places for user configuration. See [Ansible roles](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html).

## Three ways to use a role

These are alternative plays. Each assumes an existing local `webserver` role and an inventory group named `webservers`. Replace `webserver` with an installed role FQCN when using collection content.

### Play-level `roles`

```yaml
- name: Configure web servers
  hosts: webservers
  roles:
    - role: webserver
      vars:
        webserver_port: 8080
```

### Dynamic `include_role`

The role is included when execution reaches this task. Conditions and loops on the include control whether/how it is included; options intended for its child tasks may need `apply`.

```yaml
- name: Include a role at a chosen point
  hosts: webservers
  tasks:
    - name: Run the webserver role
      ansible.builtin.include_role:
        name: webserver
      vars:
        webserver_port: 8080
```

See [`include_role`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/include_role_module.html).

### Static `import_role`

The role's tasks are expanded during playbook parsing and execute at this position in the task list. Conditions and tags on an import are applied to the imported tasks; imports and includes therefore behave differently.

```yaml
- name: Import a role at a chosen point
  hosts: webservers
  tasks:
    - name: Import the webserver role
      ansible.builtin.import_role:
        name: webserver
      vars:
        webserver_port: 8080
```

See [`import_role`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/import_role_module.html).

## Play execution order

The usual successful sequence is:

1. Fact gathering, if enabled.
2. `pre_tasks`, then their notified handlers.
3. Play-level `roles`, with dependencies processed before dependent roles.
4. `tasks`, including any roles imported/included there.
5. Handlers notified by roles/tasks.
6. `post_tasks`, then their notified handlers.

Handlers can also run earlier with `meta: flush_handlers`. Conditions, tags, failures, and duplicate-role rules affect what actually runs. This is a play structure, not a guarantee that every host completes every role before any host starts ordinary tasks: `serial` batches and [execution strategy](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_strategies.html) matter. See [role execution order](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html#using-roles-at-the-play-level).

## SSH and file transfer

Ansible's connection plugin controls transport. For `ansible.builtin.ssh`, the default transfer method is `smart`: try SFTP, then SCP, then a piped transfer. It is more precise to say “SFTP is tried first” than “Ansible always uses SFTP.” SSH also executes remote commands; file transfer is only one part of the connection workflow.

To select SFTP explicitly in `ansible.cfg`:

```ini
[ssh_connection]
transfer_method = sftp
```

Legacy SCP can be selected with `transfer_method = scp`; the documented OpenSSH 9+ compatibility setting is `scp_extra_args = -O`. Use this only when the legacy protocol is actually required. See [SSH connection options](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/ssh_connection.html).

`ansible.posix.synchronize` wraps rsync for efficient directory/file synchronization. It belongs to the `ansible.posix` collection, not `ansible.builtin`, and requires rsync on the originating and receiving hosts. It normally originates from the controller, but delegation can change that. Transfers still depend on authentication, paths, and permissions; it is not simply an automatic “large-file mode.” See [`synchronize`](https://docs.ansible.com/ansible/latest/collections/ansible/posix/synchronize_module.html).

## Source provenance

<!-- Org properties: {"craft_id": "7089CF11-90D2-4774-81DB-147B28EFE7D4", "imported": "2026-09-27"} -->

Original import: [Ansible in Craft](craftdocs://open?spaceId=f0e27734-d8b8-47ce-be9d-9b35def0cb70&blockId=7089CF11-90D2-4774-81DB-147B28EFE7D4), imported 2026-09-27. Duplicate imported content has been consolidated above. Reviewed against the linked documentation on 2026-09-30.
