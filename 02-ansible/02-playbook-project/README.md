# Step 2: Creating a Playbook Project

## Objective

Scaffold a playbook project using the VS Code Ansible extension, then write a simple playbook that automates workspace setup using only `ansible.builtin` modules. A playbook project is the most basic Ansible artifact — the foundation everything else in this module builds upon.

## Prerequisites

- Completed Step 1 (Introduction)
- Dev Spaces workspace running

---

## Step 2.1: Playbook projects — the basic Ansible artifact

A **playbook project** is the simplest unit of Ansible automation. It contains:

| File | Purpose |
|---|---|
| `site.yml` | Entry-point playbook describing what to automate |
| `inventory/hosts.yml` | Static inventory declaring *where* to automate |
| `inventory/group_vars/` | Variables that configure *how* the automation behaves |
| `requirements.yml` | Collection dependencies (empty for now) |
| `ansible.cfg` | Project-scoped Ansible configuration |

`ansible-creator init playbook` scaffolds a project following the [Ansible best practices directory layout](https://docs.ansible.com/ansible/latest/tips_tricks/sample_setup.html) so you never have to figure out the structure from scratch.

---

## Step 2.2: Scaffold the playbook project via the VS Code extension

Using the VS Code extension is the recommended path — it provides a guided wizard and calls `ansible-creator` under the hood.

1. Click the **Ansible extension icon** in the left sidebar (the "A" logo)
2. Under **INITIALIZE**, click **Playbook project**
3. Fill in the form:

| Field | Value |
|---|---|
| Namespace | `acme` |
| Collection | `acmecollection` |
| Destination directory | `/projects/ansible-dev-tools-workspace/acmecorp-playbook` |

4. Click **Create**

**CLI equivalent:**

Option A — create the project inside an existing directory:

```bash
cd /projects/ansible-dev-tools-workspace
ansible-creator init playbook acme.acmecollection
```

Option B — specify the destination directory explicitly:

```bash
ansible-creator init playbook acme.acmecollection \
  /projects/ansible-dev-tools-workspace/acmecorp-playbook
```

> `acme.acmecollection` is the `<namespace>.<name>` collection name (matches the values entered in the wizard above). The path argument is the destination directory; when omitted, the project is created in the current working directory.

<details>
<summary>✅ Verification: Project scaffolded</summary>

```bash
ls /projects/ansible-dev-tools-workspace/acmecorp-playbook/
```

**Expected:**
```
ansible.cfg   inventory/   requirements.yml   site.yml
```

</details>

---

## Step 2.3: Define the inventory

Replace `inventory/hosts.yml` with a minimal localhost inventory:

```yaml
---
all:
  hosts:
    localhost:
      ansible_connection: local
```

For a real environment, replace `localhost` with your managed hosts:

```yaml
---
all:
  children:
    dev_servers:
      hosts:
        server1.example.com:
        server2.example.com:
```

<details>
<summary>✅ Verification: Inventory parsed correctly</summary>

```bash
cd /projects/ansible-dev-tools-workspace/acmecorp-playbook
ansible-inventory -i inventory/ --list
```

</details>

---

## Step 2.4: Set group variables

Create `inventory/group_vars/all.yml` to declare variables for all hosts. Keeping variables in `group_vars/` — rather than hardcoded in the playbook — makes it easy to override them per environment without touching the playbook code.

```yaml
---
# Base directory where the workspace will be created
role_acmecorp_setup_base: /tmp/acme-workspace

# Subdirectories to create within the workspace
role_acmecorp_setup_dirs:
  - projects
  - logs
  - configs
```

<details>
<summary>✅ Verification: Variables file is valid YAML</summary>

```bash
python3 -c "import yaml; yaml.safe_load(open('inventory/group_vars/all.yml'))" && echo "Valid YAML"
```

</details>

---

## Step 2.5: Write the playbook

Replace `site.yml` with a playbook that creates a structured workspace directory. This playbook uses **only `ansible.builtin` modules** — no external collections required yet.

```yaml
---
- name: Set up development workspace
  hosts: all
  become: false

  vars:
    role_acmecorp_setup_base: /tmp/acme-workspace
    role_acmecorp_setup_dirs:
      - projects
      - logs
      - configs

  tasks:
    - name: Create workspace base directory
      ansible.builtin.file:
        path: "{{ role_acmecorp_setup_base }}"
        state: directory
        mode: "0755"

    - name: Create workspace subdirectories
      ansible.builtin.file:
        path: "{{ role_acmecorp_setup_base }}/{{ item }}"
        state: directory
        mode: "0755"
      loop: "{{ role_acmecorp_setup_dirs }}"

    - name: Create default configuration file
      ansible.builtin.copy:
        content: |
          # Workspace configuration
          role_acmecorp_setup_base: {{ role_acmecorp_setup_base }}
          created_by: Ansible
        dest: "{{ role_acmecorp_setup_base }}/configs/settings.yml"
        mode: "0644"

    - name: Report workspace status
      ansible.builtin.debug:
        msg: "Acmespace ready at {{ role_acmecorp_setup_base }}"
```

> **Why `ansible.builtin` only?** This keeps Step 2 completely self-contained — no collection installation needed. In Step 3 you will extract these tasks into a role, and in Step 4 you will package that role into a collection.

<details>
<summary>✅ Verification: Playbook lints cleanly</summary>

```bash
cd /projects/ansible-dev-tools-workspace/acmecorp-playbook
ansible-lint site.yml
```

</details>

---

## Step 2.6: Run the playbook

> **Note:** In AAP 2.x, `ansible-navigator run` is the recommended way to execute playbooks. It mirrors exactly how automation controller runs them — through an Execution Environment — making local runs and production runs consistent. `ansible-playbook` still works but bypasses the EE layer.

```bash
cd /projects/ansible-dev-tools-workspace/acmecorp-playbook
ansible-navigator run site.yml -i inventory/ --mode stdout --ee false
```

The `--ee false` flag tells `ansible-navigator` to use the locally installed `ansible-core` instead of pulling an EE container image, which is the right choice inside Dev Spaces where the tools are already installed.

**Fallback** (if `ansible-navigator` is not available):

```bash
ansible-playbook -i inventory/ site.yml
```

<details>
<summary>✅ Verification: Playbook succeeds</summary>

**Expected output:**
```
PLAY [Set up development workspace] *******************************************

TASK [Gathering Facts] ********************************************************
ok: [localhost]

TASK [Create workspace base directory] ****************************************
changed: [localhost]

TASK [Create workspace subdirectories] ****************************************
changed: [localhost] => (item=projects)
changed: [localhost] => (item=logs)
changed: [localhost] => (item=configs)

TASK [Create default configuration file] **************************************
changed: [localhost]

TASK [Report workspace status] ************************************************
ok: [localhost] => {
    "msg": "Acmespace ready at /tmp/acme-workspace"
}

PLAY RECAP ********************************************************************
localhost : ok=5  changed=4  unreachable=0  failed=0
```

Verify the workspace was created:

```bash
ls /tmp/acme-workspace/
```

**Expected:**
```
configs/  logs/  projects/
```

</details>

---

## Summary

You have successfully:

1. Scaffolded a playbook project using the VS Code extension and `ansible-creator`
2. Defined a localhost inventory
3. Set variables in `group_vars/all.yml` following best practices
4. Written a clean playbook using only `ansible.builtin` modules
5. Ran the playbook and verified the results

## Project structure

```
acmecorp-playbook/
├── ansible.cfg
├── site.yml                         ← workspace setup tasks (inline for now)
├── requirements.yml                 ← empty (no external collections yet)
└── inventory/
    ├── hosts.yml                    ← localhost inventory
    └── group_vars/
        └── all.yml                  ← role_acmecorp_setup_base, role_acmecorp_setup_dirs, role_acmecorp_setup_config_file
```

## Next Step

Proceed to [03-role-project](../03-role-project/) to extract the playbook tasks into a reusable role and add it back to this playbook project.

## References

- [ansible-creator Documentation](https://ansible.readthedocs.io/projects/creator/)
- [Ansible best practices layout](https://docs.ansible.com/ansible/latest/tips_tricks/sample_setup.html)
- [ansible.builtin modules](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/index.html)
