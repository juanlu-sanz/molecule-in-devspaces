# Step 3: Creating a Role Project

## Objective

Scaffold a standalone `role_acmecorp_setup` role using `ansible-galaxy role init`, implement it following Ansible best practices, and add it to the playbook project from Step 2. A role packages reusable automation logic into a standard structure that can be shared, tested with Molecule, and later packaged inside a collection.

## Prerequisites

- Completed Step 2 (`acmecorp-playbook` project exists and runs)
- Dev Spaces workspace running

---

## Step 3.1: Roles — reusable automation units

A **role** is a structured way to package and reuse Ansible automation. It follows a standard directory layout that the community recognizes:

```
role_acmecorp_setup/
├── defaults/main.yml   ← default variables (overridable by callers)
├── tasks/main.yml      ← the actual automation steps
├── handlers/main.yml   ← event-driven tasks (e.g. restart a service)
├── meta/main.yml       ← role metadata (dependencies, platform support)
├── vars/main.yml       ← role-internal variables (not overridable)
├── files/              ← static files to copy to managed hosts
└── templates/          ← Jinja2 templates
```

Roles are better than inline tasks because:

| Benefit | Why it matters |
|---|---|
| **Reusable** | The same role can be called from any number of playbooks |
| **Testable** | Molecule (Module 3) tests roles in isolated environments |
| **Consistent structure** | Any Ansible developer immediately understands the layout |
| **Shareable** | Roles can be published to Ansible Galaxy |
| **Composable** | Roles are the building block inside collections (Step 4) |

---

## Step 3.2: Scaffold the role

Create a `roles/` directory in your workspace and initialize the role using the VS Code extension:

1. Click the **Ansible extension icon** in the left sidebar (the "A" logo)
2. Under **INITIALIZE**, click **Role**
3. Fill in the form:

| Field | Value |
|---|---|
| Role name | `role_acmecorp_setup` |
| Destination directory | `/projects/ansible-dev-tools-workspace/roles` |

4. Click **Create**

> **CLI alternative:**
> ```bash
> mkdir -p /projects/ansible-dev-tools-workspace/roles
> cd /projects/ansible-dev-tools-workspace/roles
> ansible-galaxy role init role_acmecorp_setup
> ```

<details>
<summary>✅ Verification: Role scaffolded</summary>

```bash
find /projects/ansible-dev-tools-workspace/roles/role_acmecorp_setup -type f | sort
```

**Expected:**
```
roles/role_acmecorp_setup/README.md
roles/role_acmecorp_setup/defaults/main.yml
roles/role_acmecorp_setup/files/.gitkeep
roles/role_acmecorp_setup/handlers/main.yml
roles/role_acmecorp_setup/meta/main.yml
roles/role_acmecorp_setup/tasks/main.yml
roles/role_acmecorp_setup/templates/.gitkeep
roles/role_acmecorp_setup/vars/main.yml
```

Then confirm the skeleton files are present and lint-clean:

```bash
ansible-lint /projects/ansible-dev-tools-workspace/roles/role_acmecorp_setup/handlers/main.yml \
             /projects/ansible-dev-tools-workspace/roles/role_acmecorp_setup/vars/main.yml
```

No errors means the scaffolded files are valid YAML.

</details>

---

## Step 3.3: Define default variables

Open `roles/role_acmecorp_setup/defaults/main.yml` and replace its contents:

```yaml
---
# Base directory where the workspace will be created
role_acmecorp_setup_base: /tmp/acme-workspace

# Subdirectories to create within the workspace
role_acmecorp_setup_dirs:
  - projects
  - logs
  - configs

# Name of the configuration file to generate
role_acmecorp_setup_config_file: settings.yml
```

All variables are prefixed with `role_acmecorp_setup_` — the **role name** — following the [Ansible best practice for role variable naming](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html#role-variable-scoping) and the convention enforced by `ansible-lint` (`var-naming[no-role-prefix]`).

> **Why the role name prefix?** Ansible variables are global within a play. If two roles both define a variable called `base` or `config_file`, the second role silently overwrites the first. Prefixing every variable with the role name (`role_acmecorp_setup_*`) makes the scope unambiguous — it is immediately obvious which role owns each variable, and collisions become impossible regardless of how many roles run together. This is the same convention used by [linux-system-roles](https://linux-system-roles.github.io/documentation/role-design-guide.html).

Using `defaults/` (not `vars/`) means callers **can override** these values. This is a best practice for reusable roles: safe defaults that work out of the box, but fully configurable.

<details>
<summary>✅ Verification: Defaults are valid YAML</summary>

```bash
ansible-lint /projects/ansible-dev-tools-workspace/roles/role_acmecorp_setup/defaults/main.yml
```

No errors means the file is valid YAML and follows Ansible conventions.

</details>

---

## Step 3.4: Write the role tasks

Open `roles/role_acmecorp_setup/tasks/main.yml` and replace its contents. These are the same tasks from Step 2, now properly packaged as a role:

```yaml
---
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
    dest: "{{ role_acmecorp_setup_base }}/configs/{{ role_acmecorp_setup_config_file }}"
    mode: "0644"

- name: Report workspace status
  ansible.builtin.debug:
    msg: "Acmespace ready at {{ role_acmecorp_setup_base }}"
```

<details>
<summary>✅ Verification: Tasks lint cleanly</summary>

```bash
ansible-lint /projects/ansible-dev-tools-workspace/roles/role_acmecorp_setup/tasks/main.yml
```

</details>

---

## Step 3.5: Update role metadata

Open `roles/role_acmecorp_setup/meta/main.yml` and set accurate metadata:

```yaml
---
galaxy_info:
  role_name: role_acmecorp_setup
  author: acme
  description: Creates a structured workspace directory with configurable subdirectories
  license: MIT
  min_ansible_version: "2.15"
  platforms:
    - name: EL
      versions:
        - "9"

dependencies: []
```

> **License note:** `ansible-galaxy role init` generates a `license` placeholder (`"license (GPL-2.0-or-later, MIT, etc.)"`) that ansible-lint rejects because it is not a valid [SPDX identifier](https://spdx.org/licenses/). Always replace it with a real value — common choices for Ansible content are `MIT`, `Apache-2.0`, and `GPL-2.0-or-later`.

Accurate metadata makes the role discoverable when published to Ansible Galaxy and provides useful context to anyone consuming it.

<details>
<summary>✅ Verification: Metadata is valid</summary>

```bash
ansible-lint /projects/ansible-dev-tools-workspace/roles/role_acmecorp_setup/meta/main.yml
```

No errors means the metadata is valid YAML, uses a proper SPDX licence identifier, and passes the `meta-incorrect` checks.

</details>

---

## Step 3.6: Add the role to the playbook project

Rather than overwriting the playbook from Step 2, rename it first so both approaches stay available side by side. This lets you run either version at any time and directly compare how the same automation can be expressed with inline tasks versus a role — two valid Ansible patterns, each with its own trade-offs.

**Rename the existing inline-tasks playbook:**

```bash
mv /projects/ansible-dev-tools-workspace/site.yml \
   /projects/ansible-dev-tools-workspace/site-standalone.yml
```

**Create a new `acmecorp-playbook/site.yml`** that delegates to the role instead:

```yaml
---
- name: Set up development workspace
  hosts: all
  become: false

  roles:
    - role: role_acmecorp_setup
```

> **Why keep both?** `site-standalone.yml` shows the *inline tasks* approach — everything visible in one file, good for simple one-off automation. `site.yml` shows the *role* approach — logic encapsulated, reusable, and testable with Molecule. As the workshop progresses, the role gets packaged into a collection (Step 4) and then into an EE (Step 5), so you can look back at `site-standalone.yml` to appreciate how much structure each layer adds.

**Update `acmecorp-playbook/ansible.cfg`** to tell Ansible where to find the role:

```ini
[defaults]
inventory = inventory/
roles_path = /projects/ansible-dev-tools-workspace/roles
```

The `roles_path` points to the directory containing `role_acmecorp_setup/`. Ansible resolves the bare role name `role_acmecorp_setup` from this path.

<details>
<summary>✅ Verification: Both playbooks lint cleanly</summary>

```bash
cd /projects/ansible-dev-tools-workspace/acmecorp-playbook
ansible-lint site.yml
```

</details>

---

## Step 3.7: Run the updated playbook

```bash
cd /projects/ansible-dev-tools-workspace/acmecorp-playbook
ansible-navigator run site.yml -i inventory/ --mode stdout --ee false
```

You can also run the standalone version to confirm both produce the same result:

```bash
ansible-navigator run site-standalone.yml -i inventory/ --mode stdout --ee false
```

<details>
<summary>✅ Verification: Role runs correctly</summary>

**Expected output (the role name now appears in each task):**
```
PLAY [Set up development workspace] *******************************************

TASK [Gathering Facts] ********************************************************
ok: [localhost]

TASK [role_acmecorp_setup : Create workspace base directory] **********************
ok: [localhost]

TASK [role_acmecorp_setup : Create workspace subdirectories] **********************
ok: [localhost] => (item=projects)
ok: [localhost] => (item=logs)
ok: [localhost] => (item=configs)

TASK [role_acmecorp_setup : Create default configuration file] ********************
ok: [localhost]

TASK [role_acmecorp_setup : Report acmecorp status] ******************************
ok: [localhost] => {
    "msg": "Acmespace ready at /tmp/acme-workspace"
}

PLAY RECAP ********************************************************************
localhost : ok=5  changed=0  unreachable=0  failed=0
```

The role name `role_acmecorp_setup :` now prefixes each task, showing that the logic is properly encapsulated. The result is identical to Step 2 — the difference is that the role can now be reused from any playbook.

</details>

---

## Summary

You have successfully:

1. Scaffolded a role using `ansible-galaxy role init`
2. Defined role variables in `defaults/main.yml` with a namespacing convention
3. Implemented the role tasks (refactored from Step 2 inline tasks)
4. Set accurate role metadata in `meta/main.yml`
5. Preserved the original inline-tasks playbook as `site-standalone.yml`
6. Created a new `site.yml` that delegates to the role
7. Verified both playbooks produce identical results

## Project structure after this step

```
acmecorp-playbook/
├── ansible.cfg                      ← roles_path added
├── site.yml                         ← role approach (role_acmecorp_setup via roles_path)
├── site-standalone.yml              ← inline tasks approach (Step 2 original)
├── requirements.yml
└── inventory/
    ├── hosts.yml
    └── group_vars/
        └── all.yml

roles/role_acmecorp_setup/
├── defaults/main.yml   ← role_acmecorp_setup_base, role_acmecorp_setup_dirs, role_acmecorp_setup_config_file
├── tasks/main.yml      ← create base → create subdirs → create config → report
└── meta/main.yml       ← author, description, license, platforms
```

## Next Step

Proceed to [04-collection](../04-collection/) to package this role inside an Ansible collection — the standard unit for distributing automation content — and update the playbook to use the role via its Fully Qualified Collection Name (FQCN).

## References

- [ansible-galaxy role init](https://docs.ansible.com/ansible/latest/cli/ansible-galaxy.html)
- [Role directory structure](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html#role-directory-structure)
- [Role variable scoping (defaults vs vars)](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#variable-precedence-where-should-i-put-a-variable)
- [linux-system-roles design guide](https://linux-system-roles.github.io/documentation/role-design-guide.html)
