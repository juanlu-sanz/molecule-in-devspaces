# Step 2: Running the Scaffolded Tests

## Objective

Run the `integration_hello_world` scenario with the podman driver, observe the full lifecycle (including real container creation/destruction), and learn the granular commands used during development iteration.

## Prerequisites

- Completed Step 1 (Introduction) — `molecule.yml` updated to podman, `converge.yml` updated to `hosts: all`, `vars.yml` fixed
- Collection available via editable install with the venv activated:

```bash
cd /projects/ansible-dev-tools-workspace/acme.mycollection
ade install -e .
source .venv/bin/activate
unset ANSIBLE_COLLECTIONS_PATH
```

> **Dev Spaces note:** The workspace image pre-sets `ANSIBLE_COLLECTIONS_PATH` to a default location. Environment variables take precedence over `ansible.cfg` in Ansible's precedence ladder, so Molecule may ignore your `collections_path` setting and fail to find the collection. Unsetting it lets the activated venv's `ansible-core` use its own default search paths, which include the venv's site-packages where `ade install -e .` placed the editable install.

---

## Step 2.1: Explore the scenario configuration

Navigate to the `extensions/` directory where Molecule lives:

```bash
cd /projects/ansible-dev-tools-workspace/acme.mycollection/extensions
```

Inspect the scenario you updated in Step 1:

```bash
cat molecule/integration_hello_world/molecule.yml
```

**Expected output:**

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

### Understanding `molecule.yml`

**`driver`**

```yaml
driver:
  name: podman
```

Molecule will use `podman` to create and manage test containers.

**`platforms`**

```yaml
platforms:
  - name: instance
    image: registry.access.redhat.com/ubi9/ubi-init:latest
    pre_build_image: true
```

A single test container named `instance`, using a UBI9 init image. `pre_build_image: true` means Molecule pulls this image as-is (no Dockerfile build step).

**`provisioner`**

```yaml
provisioner:
  name: ansible
  playbooks:
    converge: ../utils/playbooks/converge.yml
```

Ansible runs on the **host** (workspace pod) and connects to the test container via the podman connection plugin. The converge playbook is shared across all scenarios.

**`config_options`**

```yaml
  config_options:
    defaults:
      collections_path: ${ANSIBLE_COLLECTIONS_PATH}
```

Passes the collections path so Ansible can resolve FQCNs like `acme.mycollection.sample_filter`. This only works correctly when `ANSIBLE_COLLECTIONS_PATH` has been unset (see Prerequisites above) — the Dev Spaces default value points to the system collections location and does not include the editable install in the venv.

<details>
<summary>✅ Verification: Scenario files in place</summary>

```bash
ls molecule/
```

**Expected:**
```
integration_hello_world/  utils/
```

```bash
ls molecule/utils/playbooks/
```

**Expected:**
```
converge.yml  noop.yml
```

</details>

---

## Step 2.2: Explore the shared converge playbook

```bash
cat molecule/utils/playbooks/converge.yml
```

**Expected output:**

```yaml
---
- name: Shared integration test runner
  hosts: all
  gather_facts: false

  tasks:
    - name: Load the vars
      ansible.builtin.include_vars:
        file: ../../utils/vars/vars.yml

    - name: "Integration test: {{ test_name }}"
      ansible.builtin.include_role:
        name: "{{ test_path }}"
      vars:
        test_path: "{{ integration_tests_path }}{{ test_name }}"
        test_name: "{{ molecule_scenario_name.replace('integration_', '') }}"
```

| Variable | What it does |
|---|---|
| `hosts: all` | Targets the container(s) Molecule created |
| `molecule_scenario_name` | Injected by Molecule — e.g. `integration_hello_world` |
| `test_name` | Strips the `integration_` prefix → `hello_world` |
| `integration_tests_path` | Loaded from `vars.yml` — path to `tests/integration/targets/` |
| `test_path` | Full path to the test target role |

This means you never need to write a separate converge playbook per scenario — you only write test target tasks under `tests/integration/targets/<target_name>/tasks/main.yml`.

---

## Step 2.3: Explore the integration test target

```bash
cat /projects/ansible-dev-tools-workspace/acme.mycollection/tests/integration/targets/hello_world/tasks/main.yml
```

The scaffolded target:

```yaml
---
- name: Test the Hello World filter plugin
  ansible.builtin.set_fact:
    msg: "{{ 'ansible-creator' | acme.mycollection.sample_filter }}"

- name: Assert that the filter worked
  ansible.builtin.assert:
    that:
      - msg == 'Hello, ansible-creator'
```

The test pattern is always: **execute → register/set_fact → assert**.

<details>
<summary>✅ Verification: Target exists</summary>

```bash
ls /projects/ansible-dev-tools-workspace/acme.mycollection/tests/integration/targets/
```

**Expected:**
```
hello_world/
```

</details>

---

## Step 2.4: Run the full lifecycle (`molecule test`)

```bash
cd /projects/ansible-dev-tools-workspace/acme.mycollection/extensions
molecule test -s integration_hello_world
```

Watch the stages execute. With the podman driver you will see:
- **create** — Molecule pulls the image and starts a container
- **converge** — Ansible connects to the container and runs the test target
- **idempotency** — converge runs again, asserting no changes
- **destroy** — the container is removed

<details>
<summary>✅ Verification: All stages pass</summary>

**Expected key output:**
```
INFO     Running integration_hello_world > create
...
TASK [Wait for instance(s) creation to complete] ******************************
changed: [localhost] => (item=instance)
...
TASK [Assert that the filter worked] ******************************************
ok: [instance]

PLAY RECAP *********************************************************************
instance                   : ok=2    changed=0    unreachable=0    failed=0

INFO     Running integration_hello_world > destroy
...
INFO     Pruning extra files from scenario ephemeral directory
```

The container name `instance` appears in the play recap (not `localhost`) — confirming the role ran inside the podman container.

</details>

---

## Step 2.5: Iterate with granular commands (development workflow)

`molecule test` destroys the environment after every run — slow for active development. Instead, **keep the container alive** and re-run only the stage you changed:

| Command | What it does |
|---|---|
| `molecule create` | Pulls image, starts the test container |
| `molecule converge` | Re-applies the role inside the container |
| `molecule verify` | Re-runs only the verify playbook |
| `molecule login` | Opens an interactive shell inside the container |
| `molecule list` | Shows container status |
| `molecule destroy` | Stops and removes the container |

```bash
cd /projects/ansible-dev-tools-workspace/acme.mycollection/extensions

# Start the test container
molecule create -s integration_hello_world

# Re-apply after each code change
molecule converge -s integration_hello_world

# Open a shell inside the container to inspect state
molecule login -s integration_hello_world

# Clean up when done
molecule destroy -s integration_hello_world
```

<details>
<summary>✅ Verification: Container is running after create</summary>

```bash
molecule list -s integration_hello_world
```

**Expected:**
```
Instance Name  Driver Name  Provisioner Name  Scenario Name            Created  Converged
instance       podman       ansible           integration_hello_world  true     false
```

Also confirm with podman directly:

```bash
podman ps --filter name=instance
```

You should see the running container.

</details>

---

## Summary

You have successfully:

1. Explored the `molecule.yml` configuration with the podman driver
2. Understood the shared `converge.yml` and how it targets the test container
3. Run the full `molecule test` lifecycle with real container creation and destruction
4. Used granular commands (`create`, `converge`, `login`, `destroy`) for development iteration
5. Confirmed that tests execute inside the container (not on localhost)

## Next Step

Proceed to [03-workspace-role](../03-workspace-role/) to write your own Molecule scenario that tests the `role_acmecorp_setup` role from Module 2.

## References

- [Molecule Configuration Reference](https://ansible.readthedocs.io/projects/molecule/configuration/)
- [ansible.builtin.assert](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/assert_module.html)
