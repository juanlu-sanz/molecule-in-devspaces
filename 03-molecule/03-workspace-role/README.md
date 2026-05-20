# Step 3: Testing the role_acmecorp_setup Role

## Objective

Write a Molecule scenario that tests the `role_acmecorp_setup` role you created in Module 2. The role creates directories and a configuration file — Molecule will create a fresh container, apply the role, verify the results, and destroy the container.

## Prerequisites

- Completed Steps 1–2 of this module
- `role_acmecorp_setup` role exists at `acme.mycollection/roles/role_acmecorp_setup/`
- Collection editable install active:

```bash
cd /projects/ansible-dev-tools-workspace/acme.mycollection
ade install -e .
```

### Confirm changes from Step 1

```bash
grep dirname extensions/molecule/utils/vars/vars.yml
grep "hosts: all" extensions/molecule/utils/playbooks/converge.yml
```

Both should return matches. If not, go back to Step 1 and apply the fixes.

---

## Step 3.1: What we are testing

The `role_acmecorp_setup` role (from Module 2, Step 3) does exactly three things:

1. Creates a base directory (`role_acmecorp_setup_base`, default `/tmp/acme-workspace`)
2. Creates subdirectories inside it (`role_acmecorp_setup_dirs`: `projects`, `logs`, `configs`)
3. Creates a configuration file at `configs/<role_acmecorp_setup_config_file>`

The integration test will:
- **Converge** — run the role inside a fresh UBI9 container
- **Verify** — assert each directory and the config file exist with the correct attributes
- **Idempotency** — run the role again and confirm nothing changes

Because the role runs inside a container, the test is fully isolated — no artifacts left on the workspace pod.

---

## Step 3.2: Create the integration test target

Create the target directory and test tasks:

```bash
mkdir -p /projects/ansible-dev-tools-workspace/acme.mycollection/tests/integration/targets/role_acmecorp_setup/tasks
```

Create `tests/integration/targets/role_acmecorp_setup/tasks/main.yml`:

```yaml
---
- name: Run role_acmecorp_setup role
  ansible.builtin.include_role:
    name: acme.mycollection.role_acmecorp_setup
  vars:
    role_acmecorp_setup_base: /tmp/acme-workspace
    role_acmecorp_setup_dirs:
      - projects
      - logs
      - configs
    role_acmecorp_setup_config_file: settings.yml

- name: Check base directory exists
  ansible.builtin.stat:
    path: /tmp/acme-workspace
  register: base_dir

- name: Assert base directory is a directory
  ansible.builtin.assert:
    that:
      - base_dir.stat.exists
      - base_dir.stat.isdir
    fail_msg: "Base directory /tmp/acme-workspace was not created"

- name: Check each subdirectory exists
  ansible.builtin.stat:
    path: "/tmp/acme-workspace/{{ item }}"
  register: subdirs
  loop:
    - projects
    - logs
    - configs

- name: Assert all subdirectories are directories
  ansible.builtin.assert:
    that:
      - item.stat.exists
      - item.stat.isdir
    fail_msg: "Subdirectory {{ item.item }} was not created"
  loop: "{{ subdirs.results }}"

- name: Check configuration file exists
  ansible.builtin.stat:
    path: /tmp/acme-workspace/configs/settings.yml
  register: config_file

- name: Assert configuration file is a file
  ansible.builtin.assert:
    that:
      - config_file.stat.exists
      - config_file.stat.isreg
    fail_msg: "Configuration file settings.yml was not created"

- name: Read configuration file content
  ansible.builtin.slurp:
    src: /tmp/acme-workspace/configs/settings.yml
  register: config_content

- name: Assert configuration file contains expected content
  ansible.builtin.assert:
    that:
      - "'created_by: Ansible' in (config_content.content | b64decode)"
    fail_msg: >
      Configuration file does not contain 'created_by: Ansible'.
      Actual content: {{ config_content.content | b64decode }}
```

<details>
<summary>✅ Verification: Target file created</summary>

```bash
cat /projects/ansible-dev-tools-workspace/acme.mycollection/tests/integration/targets/role_acmecorp_setup/tasks/main.yml
```

The file should contain the tasks above.

</details>

---

## Step 3.3: Create the Molecule scenario

```bash
mkdir -p /projects/ansible-dev-tools-workspace/acme.mycollection/extensions/molecule/integration_role_acmecorp_setup
```

Create `extensions/molecule/integration_role_acmecorp_setup/molecule.yml`:

```yaml
---
driver:
  name: podman

platforms:
  - name: instance
    image: registry.access.redhat.com/ubi9/ubi-init:latest
    pre_build_image: true

provisioner:
  name: ansible
  playbooks:
    cleanup: ../utils/playbooks/noop.yml
    converge: ../utils/playbooks/converge.yml
    prepare: ../utils/playbooks/noop.yml
  config_options:
    defaults:
      collections_path: ${ANSIBLE_COLLECTIONS_PATH}

verifier:
  name: ansible
```

Key points:
- **`driver: podman`** — creates a real container for the test
- **`ubi-init`** — a minimal RHEL image suitable for Ansible testing
- **`converge.yml`** — the shared entry point resolves the test target from the scenario name

<details>
<summary>✅ Verification: Scenario file created</summary>

```bash
cat /projects/ansible-dev-tools-workspace/acme.mycollection/extensions/molecule/integration_role_acmecorp_setup/molecule.yml
```

</details>

---

## Step 3.4: Run the full lifecycle

```bash
cd /projects/ansible-dev-tools-workspace/acme.mycollection/extensions
molecule test -s integration_role_acmecorp_setup
```

Watch the stages:
- **create** — pulls UBI9 image, starts a container named `instance`
- **converge** — runs `role_acmecorp_setup` inside the container, then asserts
- **idempotency** — runs converge again, confirms no changes
- **destroy** — removes the container

<details>
<summary>✅ Verification: All stages pass</summary>

**Expected key output:**
```
TASK [Assert base directory is a directory] ***********************************
ok: [instance]

TASK [Assert all subdirectories are directories] ******************************
ok: [instance] => (item={'item': 'projects', ...})
ok: [instance] => (item={'item': 'logs', ...})
ok: [instance] => (item={'item': 'configs', ...})

TASK [Assert configuration file is a file] ************************************
ok: [instance]

TASK [Assert configuration file contains expected content] ********************
ok: [instance]

PLAY RECAP *********************************************************************
instance                   : ok=X    changed=0    unreachable=0    failed=0
```

The host name `instance` confirms the role ran inside the podman container.

</details>

---

## Step 3.5: Iterate — develop with granular commands

```bash
cd /projects/ansible-dev-tools-workspace/acme.mycollection/extensions

# Start the test container (stays alive between converge runs)
molecule create -s integration_role_acmecorp_setup

# Re-apply the role
molecule converge -s integration_role_acmecorp_setup

# Shell into the container to inspect
molecule login -s integration_role_acmecorp_setup
# Inside the container:
#   ls /tmp/acme-workspace/
#   cat /tmp/acme-workspace/configs/settings.yml
#   exit

# Destroy when done
molecule destroy -s integration_role_acmecorp_setup
```

<details>
<summary>✅ Verification: Can login to container</summary>

After `molecule create`, run:

```bash
molecule login -s integration_role_acmecorp_setup
```

You should get a shell prompt inside the container. Type `exit` to return.

</details>

---

## Step 3.6: Test custom variable overrides

Create a second scenario that tests the role with different inputs:

```bash
mkdir -p /projects/ansible-dev-tools-workspace/acme.mycollection/extensions/molecule/integration_workspace_custom
mkdir -p /projects/ansible-dev-tools-workspace/acme.mycollection/tests/integration/targets/workspace_custom/tasks
```

Create `extensions/molecule/integration_workspace_custom/molecule.yml`:

```yaml
---
driver:
  name: podman

platforms:
  - name: instance
    image: registry.access.redhat.com/ubi9/ubi-init:latest
    pre_build_image: true

provisioner:
  name: ansible
  playbooks:
    cleanup: ../utils/playbooks/noop.yml
    converge: ../utils/playbooks/converge.yml
    prepare: ../utils/playbooks/noop.yml
  config_options:
    defaults:
      collections_path: ${ANSIBLE_COLLECTIONS_PATH}
  inventory:
    host_vars:
      instance:
        role_acmecorp_setup_base: /tmp/custom-acme-workspace
        role_acmecorp_setup_dirs:
          - data
          - reports
          - configs
        role_acmecorp_setup_config_file: custom.yml

verifier:
  name: ansible
```

> Note: `host_vars` key is `instance` (the container name), not `localhost`.

Create `tests/integration/targets/workspace_custom/tasks/main.yml`:

```yaml
---
- name: Run role_acmecorp_setup role with custom variables
  ansible.builtin.include_role:
    name: acme.mycollection.role_acmecorp_setup

- name: Check custom base directory exists
  ansible.builtin.stat:
    path: /tmp/custom-acme-workspace
  register: custom_base

- name: Assert custom base directory is a directory
  ansible.builtin.assert:
    that:
      - custom_base.stat.exists
      - custom_base.stat.isdir
    fail_msg: "Custom base /tmp/custom-acme-workspace was not created"

- name: Check each custom subdirectory exists
  ansible.builtin.stat:
    path: "/tmp/custom-acme-workspace/{{ item }}"
  register: custom_dirs
  loop:
    - data
    - reports
    - configs

- name: Assert custom subdirectories are directories
  ansible.builtin.assert:
    that:
      - item.stat.exists
      - item.stat.isdir
    fail_msg: "Subdirectory {{ item.item }} was not created"
  loop: "{{ custom_dirs.results }}"

- name: Check custom configuration file exists
  ansible.builtin.stat:
    path: /tmp/custom-acme-workspace/configs/custom.yml
  register: custom_config

- name: Assert custom configuration file is a file
  ansible.builtin.assert:
    that:
      - custom_config.stat.exists
      - custom_config.stat.isreg
    fail_msg: "Custom config file custom.yml was not created"
```

Run the scenario using granular commands so you can inspect the container before it's destroyed:

```bash
cd /projects/ansible-dev-tools-workspace/acme.mycollection/extensions
molecule converge -s integration_workspace_custom
```

<details>
<summary>✅ Verification: Inspect the container with podman</summary>

After converge completes, the container is still running. Use `podman exec` to verify the custom variables were applied **inside the container**:

```bash
podman exec instance ls /tmp/custom-acme-workspace/
```

**Expected:**
```
configs  data  reports
```

```bash
podman exec instance cat /tmp/custom-acme-workspace/configs/custom.yml
```

**Expected:**
```
# Workspace configuration
role_acmecorp_setup_base: /tmp/custom-acme-workspace
created_by: Ansible
```

Confirm the default path does **not** exist (proving the override worked):

```bash
podman exec instance ls /tmp/acme-workspace 2>&1
```

**Expected:** `No such file or directory`

Once verified, run the full lifecycle to confirm assertions pass end-to-end:

```bash
molecule destroy -s integration_workspace_custom
molecule test -s integration_workspace_custom
```

</details>

---

## Summary

You have successfully:

1. Written an integration test target that tests `role_acmecorp_setup` end-to-end
2. Created a Molecule scenario with the podman driver and a UBI9 container
3. Run the full lifecycle — container creation, role execution, assertions, destruction
4. Used `molecule login` to shell into the test container for debugging
5. Written a second scenario testing variable overrides

## Final collection structure

```
acme.mycollection/
├── roles/
│   └── role_acmecorp_setup/
├── extensions/
│   └── molecule/
│       ├── integration_hello_world/           ← scaffolded, updated to podman
│       ├── integration_role_acmecorp_setup/   ← default variables test
│       └── integration_workspace_custom/      ← override variables test
└── tests/
    └── integration/
        └── targets/
            ├── hello_world/
            ├── role_acmecorp_setup/
            └── workspace_custom/
```

## Next Step (Bonus)

Proceed to [04-bonus](../04-bonus/) to test a filesystem setup role that wraps `redhat.rhel_system_roles.storage`.

## References

- [Molecule Documentation](https://ansible.readthedocs.io/projects/molecule/)
- [ansible.builtin.assert](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/assert_module.html)
- [ansible.builtin.stat](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/stat_module.html)
