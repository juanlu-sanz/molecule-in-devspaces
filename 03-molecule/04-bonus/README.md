# Step 4: Bonus — Testing Filesystem Setup with RHEL System Roles

## Objective

Create a new collection (`acme.infra`) with a wrapper role (`role_fs_setup`) that uses the Red Hat certified `redhat.rhel_system_roles.storage` role for filesystem creation. Write Molecule tests that validate the role's input logic using the podman driver.

## What you will learn

- Declaring a Red Hat certified collection as a dependency
- Creating a wrapper role around a RHEL System Role
- Splitting role tasks for testability (validate vs. execute)
- Writing Molecule tests that verify input validation inside containers
- The `block`/`rescue` pattern for testing expected failures

## Prerequisites

- Completed Module 2 and Steps 1–3 of Module 3
- A Red Hat account (free Developer subscription is sufficient)

---

## Step 4.1: Obtain an Automation Hub token

`redhat.rhel_system_roles` is a Red Hat certified collection published on [Automation Hub](https://console.redhat.com/ansible/automation-hub/). You need an API token to download it.

1. Open [console.redhat.com](https://console.redhat.com/) and log in
2. Navigate to **Ansible Automation Platform** → **Automation Hub** → **Connect to Hub**
3. Click **Load token** and copy it

<details>
<summary>✅ Verification: token saved</summary>

Keep the token handy — you will paste it into `ansible.cfg` in Step 4.2.

</details>

---

## Step 4.2: Scaffold the collection

```bash
ansible-creator init collection acme.infra \
  /projects/ansible-dev-tools-workspace/acme.infra

cd /projects/ansible-dev-tools-workspace/acme.infra
```

Replace `ansible.cfg` contents:

```bash
cat > ansible.cfg << 'EOF'
[defaults]
collections_path = .

[galaxy]
server_list = automation_hub, galaxy

[galaxy_server.automation_hub]
url = https://console.redhat.com/api/automation-hub/content/published/
auth_url = https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token
token = <paste-your-token-here>

[galaxy_server.galaxy]
url = https://galaxy.ansible.com/
EOF
```

Replace `<paste-your-token-here>` with your token.

Open `galaxy.yml` and replace its contents:

```yaml
namespace: "acme"
name: "infra"
version: 1.0.0
readme: README.md
authors:
  - Your Name <your.email@example.com>

description: Infrastructure roles wrapping RHEL System Roles

dependencies:
  "redhat.rhel_system_roles": "*"

build_ignore:
  - .gitignore
  - .venv
  - collections
```

Install dependencies and activate the venv:

```bash
ade install -e .
source .venv/bin/activate
unset ANSIBLE_COLLECTIONS_PATH
```

> **Dev Spaces note:** The workspace image pre-sets `ANSIBLE_COLLECTIONS_PATH` to a default location that does not include the venv. Unsetting it lets the activated venv's `ansible-core` use its own default search paths, which include the editable install.

<details>
<summary>✅ Verification: RHEL System Roles installed</summary>

```bash
ansible-galaxy collection list | grep rhel_system_roles
```

**Expected:**
```
redhat.rhel_system_roles   X.X.X
```

</details>

---

## Step 4.3: Create the wrapper role

```bash
cd /projects/ansible-dev-tools-workspace/acme.infra
mkdir -p roles && cd roles
ansible-galaxy role init role_fs_setup
```

Fix linting issues in empty files:

```bash
echo "---" > role_fs_setup/handlers/main.yml
echo "---" > role_fs_setup/vars/main.yml
```

### defaults/main.yml

```yaml
---
role_fs_setup_device: ""
role_fs_setup_fs_type: xfs
role_fs_setup_mount_point: ""
role_fs_setup_mount_mode: "0755"
```

### tasks/validate.yml

Create `roles/role_fs_setup/tasks/validate.yml`:

```yaml
---
- name: Validate required variables are provided
  ansible.builtin.assert:
    that:
      - role_fs_setup_device | length > 0
      - role_fs_setup_mount_point | length > 0
    fail_msg: >-
      Required variables missing —
      role_fs_setup_device='{{ role_fs_setup_device }}'
      role_fs_setup_mount_point='{{ role_fs_setup_mount_point }}'.
      Both must be non-empty.

- name: Validate filesystem type is supported
  ansible.builtin.assert:
    that:
      - role_fs_setup_fs_type in ['xfs', 'ext4', 'ext3', 'ext2']
    fail_msg: >-
      Unsupported filesystem type '{{ role_fs_setup_fs_type }}'.
      Supported: xfs, ext4, ext3, ext2.

- name: Display validated storage configuration
  ansible.builtin.debug:
    msg: >-
      Storage config validated —
      {{ role_fs_setup_fs_type }} on {{ role_fs_setup_device }}
      → {{ role_fs_setup_mount_point }} (mode {{ role_fs_setup_mount_mode }})
```

### tasks/main.yml

```yaml
---
- name: Validate inputs
  ansible.builtin.include_tasks: validate.yml

- name: Configure filesystem via RHEL System Roles
  ansible.builtin.include_role:
    name: redhat.rhel_system_roles.storage
  vars:
    storage_volumes:
      - name: "{{ role_fs_setup_mount_point | basename }}"
        type: disk
        disks:
          - "{{ role_fs_setup_device | basename }}"
        fs_type: "{{ role_fs_setup_fs_type }}"
        mount_point: "{{ role_fs_setup_mount_point }}"
        mount_mode: "{{ role_fs_setup_mount_mode }}"
```

### meta/main.yml

```yaml
---
galaxy_info:
  role_name: role_fs_setup
  author: acme
  description: Wrapper role for filesystem creation using RHEL System Roles storage
  license: MIT
  min_ansible_version: "2.15"
  platforms:
    - name: EL
      versions:
        - "9"

dependencies: []
```

<details>
<summary>✅ Verification: Role structure</summary>

```bash
ansible-lint /projects/ansible-dev-tools-workspace/acme.infra/roles/role_fs_setup/
```

Should pass with no errors (warnings are acceptable).

</details>

---

## Step 4.4: Write the Molecule tests

The validation logic in `validate.yml` runs pure Ansible assertions — perfect for testing inside a container. We test three cases: missing variables, invalid filesystem type, and valid input.

### Test target

```bash
mkdir -p /projects/ansible-dev-tools-workspace/acme.infra/tests/integration/targets/fs_setup_validation/tasks
```

Create `tests/integration/targets/fs_setup_validation/tasks/main.yml`:

```yaml
---
- name: "Test: role rejects empty required variables"
  block:
    - name: Call validate.yml with empty defaults
      ansible.builtin.include_role:
        name: acme.infra.role_fs_setup
        tasks_from: validate.yml
      vars:
        role_fs_setup_device: ""
        role_fs_setup_mount_point: ""

    - name: Unreachable — validation should have failed
      ansible.builtin.fail:
        msg: "Role should have failed with empty required variables"
  rescue:
    - name: Confirm validation caught the missing variables
      ansible.builtin.debug:
        msg: "PASS — role correctly rejected empty required variables"

- name: "Test: role rejects unsupported filesystem type"
  block:
    - name: Call validate.yml with invalid fs_type
      ansible.builtin.include_role:
        name: acme.infra.role_fs_setup
        tasks_from: validate.yml
      vars:
        role_fs_setup_device: /dev/sdb
        role_fs_setup_mount_point: /mnt/data
        role_fs_setup_fs_type: btrfs

    - name: Unreachable — validation should have failed
      ansible.builtin.fail:
        msg: "Role should have failed with unsupported fs_type"
  rescue:
    - name: Confirm validation caught the bad filesystem type
      ansible.builtin.debug:
        msg: "PASS — role correctly rejected unsupported filesystem type 'btrfs'"

- name: "Test: role accepts valid variables"
  ansible.builtin.include_role:
    name: acme.infra.role_fs_setup
    tasks_from: validate.yml
  vars:
    role_fs_setup_device: /dev/sdb
    role_fs_setup_mount_point: /mnt/data
    role_fs_setup_fs_type: xfs

- name: Confirm all validation tests passed
  ansible.builtin.debug:
    msg: "ALL VALIDATION TESTS PASSED"
```

### Molecule scenario

```bash
mkdir -p /projects/ansible-dev-tools-workspace/acme.infra/extensions/molecule/integration_fs_setup_validation
```

Create `extensions/molecule/integration_fs_setup_validation/molecule.yml`:

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

<details>
<summary>✅ Verification: Scenario file created</summary>

```bash
cat /projects/ansible-dev-tools-workspace/acme.infra/extensions/molecule/integration_fs_setup_validation/molecule.yml
```

</details>

---

## Step 4.5: Run the tests

> Make sure the venv is still activated and `ANSIBLE_COLLECTIONS_PATH` is unset (see Step 4.2). If you opened a new terminal, re-run `source .venv/bin/activate && unset ANSIBLE_COLLECTIONS_PATH` first.

```bash
cd /projects/ansible-dev-tools-workspace/acme.infra/extensions
molecule test -s integration_fs_setup_validation
```

<details>
<summary>✅ Verification: All tests pass</summary>

**Expected key output:**
```
TASK [Confirm validation caught the missing variables] ************************
ok: [instance] => {
    "msg": "PASS — role correctly rejected empty required variables"
}

TASK [Confirm validation caught the bad filesystem type] **********************
ok: [instance] => {
    "msg": "PASS — role correctly rejected unsupported filesystem type 'btrfs'"
}

TASK [acme.infra.role_fs_setup : Display validated storage configuration] *****
ok: [instance] => {
    "msg": "Storage config validated — xfs on /dev/sdb → /mnt/data (mode 0755)"
}

TASK [Confirm all validation tests passed] ************************************
ok: [instance] => {
    "msg": "ALL VALIDATION TESTS PASSED"
}

PLAY RECAP *********************************************************************
instance                   : ok=12   changed=0    unreachable=0    failed=0    skipped=0    rescued=2    ignored=0
```

`rescued=2` is expected — the two intentional failures (missing variables, bad fs_type) were caught by the `rescue` blocks.

</details>

---

## Summary

You have successfully:

1. Created a new collection (`acme.infra`) with a `redhat.rhel_system_roles` dependency
2. Built a wrapper role (`role_fs_setup`) that simplifies the storage role's interface
3. Split role tasks into `validate.yml` and `main.yml` for testability
4. Written Molecule tests using `block`/`rescue` patterns inside a podman container
5. Verified that validation logic works in full container isolation

### Design pattern: validate → execute

```
tasks/
├── validate.yml   ← pure assertions, testable in containers
└── main.yml       ← validate + call system role (needs real hardware)
```

This pattern applies to any role that wraps a privileged or hardware-dependent operation. Test the contract (validation) in containers during development. Test the full execution in CI with appropriate infrastructure (VMs with block devices).

## References

- [redhat.rhel_system_roles on Automation Hub](https://console.redhat.com/ansible/automation-hub/repo/published/redhat/rhel_system_roles/)
- [RHEL System Roles — Storage](https://access.redhat.com/articles/3050101)
- [Molecule block/rescue testing pattern](https://ansible.readthedocs.io/projects/molecule/)
