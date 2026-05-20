# Step 1: Introduction to Molecule

## Objective

Understand what Molecule is, how it integrates with the collection you built in Module 2, and how its lifecycle maps to the stages of a test run.

## Prerequisites

- Completed Module 2 (`acme.mycollection` with `role_acmecorp_setup` role)
- Workspace running with the nested podman image (set up in [Module 1 Step 5](../../01-devspaces/setup/))

All tools (molecule, ansible-core, ansible-lint) are pre-installed in the workspace image. No virtual environment activation or pip install is needed.

---

## Step 1.1: What is Molecule?

Molecule is the standard testing framework for Ansible collections. It automates the repetitive cycle of:

1. **Set up** a test environment (a container)
2. **Apply** your role or playbook against it
3. **Assert** the expected state with verification tasks
4. **Clean up** the environment (destroy the container)

Without Molecule you would manually set up targets, run Ansible, check results, and clean up — every time you make a change. Molecule wraps all of this into a single command and enforces a consistent structure.

| Concept | What it is |
|---|---|
| **Scenario** | A named test case — its own configuration, converge playbook, and verify playbook |
| **Driver** | The backend that creates the test environment (this workshop uses **podman**) |
| **Lifecycle stage** | One step in the test sequence (create, converge, verify, destroy, ...) |
| **Converge** | The playbook that applies your role or tasks to the test target |
| **Verify** | The playbook that asserts the expected end state |

<details>
<summary>✅ Verification: Molecule available</summary>

```bash
molecule --version
```

**Expected output:**
```
molecule X.X.X using python 3.X.X
    ansible:X.X.X
    podman:X.X.X from molecule_plugins
```

</details>

---

## Step 1.2: The Podman Driver

This workshop uses the **podman driver** for all Molecule scenarios. Molecule creates a real container for each test platform, runs the role inside it, and destroys the container when done.

| Aspect | Detail |
|---|---|
| **Isolation** | Each test runs in a fresh container — nothing persists on the host |
| **Reproducibility** | Same container image every time, regardless of workspace state |
| **Cleanup** | Automatic — `molecule destroy` removes the container |
| **Idempotency** | Easy to verify — run converge twice in a clean container |

A typical `molecule.yml` with the podman driver:

```yaml
driver:
  name: podman

platforms:
  - name: instance
    image: registry.access.redhat.com/ubi9/ubi-init:latest
    pre_build_image: true

provisioner:
  name: ansible

verifier:
  name: ansible
```

> **How it works:** Molecule uses `podman` on the workspace pod to create a test container. Ansible (running on the host) connects to that container via the `podman` connection plugin. The role executes inside the container as if it were a remote managed node.

<details>
<summary>✅ Verification: Podman works</summary>

```bash
podman run --rm registry.access.redhat.com/ubi9/ubi-minimal echo "Podman ready"
```

**Expected:** `Podman ready`

> **Troubleshooting:** If you get `Cannot connect to Podman` or a socket error, the Podman service needs to be started. The workspace `postStart` hook does this automatically, but if it hasn't run yet or timed out, start it manually:
>
> ```bash
> unset CONTAINER_HOST
> mkdir -p /run/user/$(id -u)/podman
> podman system service --time=0 unix:///run/user/$(id -u)/podman/podman.sock &
> sleep 2
> ```
>
> Then retry the `podman run` command.

</details>

---

## Step 1.3: The Molecule lifecycle

Running `molecule test` executes all stages in order:

```
dependency    ← install collection/role requirements
    ↓
destroy       ← ensure no leftover containers from a previous run
    ↓
syntax        ← ansible-playbook --syntax-check on the converge playbook
    ↓
create        ← start the test container(s)
    ↓
prepare       ← optional pre-configuration (install packages, etc.)
    ↓
converge      ← apply the role/tasks under test
    ↓
idempotency   ← run converge again and assert nothing changed
    ↓
verify        ← run assertions against the container state
    ↓
cleanup       ← optional post-test cleanup inside the container
    ↓
destroy       ← stop and remove the container(s)
```

During development you do **not** run the full lifecycle every time — that would be slow. Instead, you use individual stage commands to iterate quickly:

```bash
molecule create    # start the test container
molecule converge  # re-apply role after each change (fast)
molecule verify    # re-run assertions after each change (fast)
molecule login     # open a shell inside the test container
molecule destroy   # stop and remove the container
```

<details>
<summary>✅ Verification: Understand the lifecycle</summary>

You can see the exact sequence Molecule will execute for a scenario:

```bash
cd /projects/ansible-dev-tools-workspace/acme.mycollection/extensions
molecule matrix test -s integration_hello_world
```

**Expected output:**
```
dependency
destroy
syntax
create
prepare
converge
idempotency
side_effect
verify
cleanup
destroy
```

</details>

---

## Step 1.4: Where Molecule lives in the collection

`ansible-creator` already generated a Molecule scenario when you scaffolded `acme.mycollection` in Module 2. The collection layout places all testing infrastructure under `extensions/`:

```
acme.mycollection/
├── extensions/
│   └── molecule/
│       ├── integration_hello_world/     ← scaffolded scenario
│       │   └── molecule.yml             ← scenario configuration
│       └── utils/                       ← shared playbooks and vars
│           ├── playbooks/
│           │   ├── converge.yml         ← shared converge entry point
│           │   └── noop.yml             ← no-op for unused lifecycle stages
│           └── vars/
│               └── vars.yml             ← paths and naming resolution
└── tests/
    └── integration/
        └── targets/
            └── hello_world/             ← test tasks for integration_hello_world
                └── tasks/
                    └── main.yml
```

**The naming convention** connects scenarios to test targets:

| Scenario directory | Target directory |
|---|---|
| `molecule/integration_hello_world/` | `tests/integration/targets/hello_world/` |
| `molecule/integration_role_acmecorp_setup/` | `tests/integration/targets/role_acmecorp_setup/` |

The `converge.yml` playbook calls the target by stripping the `integration_` prefix from the scenario name.

<details>
<summary>✅ Verification: Scaffolded structure exists</summary>

```bash
cd /projects/ansible-dev-tools-workspace/acme.mycollection
ls extensions/molecule/
ls tests/integration/targets/
```

You should see `integration_hello_world` in the first output and `hello_world` in the second.

</details>

### Update `requirements.txt` for the podman driver

The collection's `requirements.txt` declares Python packages that `ade install -e .` installs into the `.venv`. The podman driver must be listed here so it's available inside the venv:

Open `/projects/ansible-dev-tools-workspace/acme.mycollection/requirements.txt` and replace its contents:

```
molecule
molecule-plugins[podman]
```

Also add `containers.podman` to `galaxy.yml` dependencies — the driver's internal playbooks use this collection:

```yaml
dependencies:
  "ansible.utils": "*"
  "ansible.posix": "*"
  "community.general": "*"
  "containers.podman": "*"
```

Then reinstall:

```bash
cd /projects/ansible-dev-tools-workspace/acme.mycollection
ade install -e .
source .venv/bin/activate
```

<details>
<summary>✅ Verification: podman driver available</summary>

```bash
molecule drivers
```

**Expected:** `podman` appears in the list.

</details>

### Update `molecule.yml` to use the podman driver

The scaffolded `molecule.yml` uses the delegated driver (no container). Update it to use podman:

Open `/projects/ansible-dev-tools-workspace/acme.mycollection/extensions/molecule/integration_hello_world/molecule.yml` and replace its contents:

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

### Update `converge.yml` to target the container

Open `/projects/ansible-dev-tools-workspace/acme.mycollection/extensions/molecule/utils/playbooks/converge.yml` and replace its contents entirely:

```yaml
---
- name: Shared integration test runner
  hosts: all
  gather_facts: false
  vars:
    collection_root: "{{ lookup('env', 'MOLECULE_SCENARIO_DIRECTORY') | dirname | dirname | dirname }}"
    test_name: "{{ lookup('env', 'MOLECULE_SCENARIO_NAME') | replace('integration_', '') }}"
    integration_tests_path: "{{ collection_root }}/tests/integration/targets/"

  tasks:
    - name: "Integration test: {{ test_name }}"
      ansible.builtin.include_role:
        name: "{{ integration_tests_path }}{{ test_name }}"
```

Key changes from the scaffold:

1. **`hosts: all`** — targets the container Molecule created (the scaffold uses `localhost`, which bypasses the container entirely)
2. **`MOLECULE_SCENARIO_DIRECTORY`** — this env var always points to the actual scenario directory (e.g. `extensions/molecule/integration_hello_world/`), regardless of where you invoke `molecule` from. Three `dirname` filters navigate up to the collection root. The scaffold uses `MOLECULE_PROJECT_DIRECTORY` which changes depending on invocation point and breaks.
3. **`lookup('env', 'MOLECULE_SCENARIO_NAME')`** — lookups execute on the **controller** (workspace pod), resolving the scenario name even when the play targets a remote container.
4. **Play-level `vars:`** — the scaffolded version uses `include_vars` from a separate file, but that causes template parsing errors when the file contains `lookup()` expressions. Inlining the vars avoids this.

> **Note:** The `utils/vars/vars.yml` file is no longer used by this converge playbook. You can delete it or leave it — it won't affect execution.

<details>
<summary>✅ Verification: converge.yml is correct</summary>

```bash
grep "hosts: all" /projects/ansible-dev-tools-workspace/acme.mycollection/extensions/molecule/utils/playbooks/converge.yml
grep "MOLECULE_SCENARIO_DIRECTORY" /projects/ansible-dev-tools-workspace/acme.mycollection/extensions/molecule/utils/playbooks/converge.yml
```

**Expected:** both return matches.

</details>

---

## Summary

You have learned:

1. What Molecule does and why collections use it
2. The podman driver — full container isolation for each test
3. The full Molecule lifecycle and which stages matter during development
4. How the scaffolded scenario structure inside `acme.mycollection` is organized
5. The naming convention linking scenarios to integration test targets

## Next Step

Proceed to [02-running-tests](../02-running-tests/) to run the `integration_hello_world` scenario with the podman driver and observe the full lifecycle in action.

## References

- [Molecule Documentation](https://ansible.readthedocs.io/projects/molecule/)
- [Molecule Configuration Reference](https://ansible.readthedocs.io/projects/molecule/configuration/)
- [Molecule Podman Driver](https://ansible.readthedocs.io/projects/molecule-plugins/podman/)
