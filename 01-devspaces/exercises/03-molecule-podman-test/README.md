# Exercise 3: Run Molecule with the Podman Driver

## Objective

Run a simple Ansible role test using Molecule with the **podman driver** — spinning up a real container inside the Dev Spaces workspace on the cluster.

## Prerequisites

- [Setup Step 5](../../setup/) completed (nested podman SCC applied on the cluster)
- A workspace created from `https://github.com/juanlu-sanz/devspaces-nested-podman`
- Working from the **workspace terminal** (not your local machine)

---

## Part A: Verify Nested Podman Works

From the workspace terminal, confirm that containers can run:

```bash
unset CONTAINER_HOST
podman run --rm registry.access.redhat.com/ubi9/ubi-minimal echo "Podman works inside Dev Spaces!"
```

<details>
<summary>✅ Verification: Container ran successfully</summary>

**Expected output (last line):**
```
Podman works inside Dev Spaces!
```

If this fails, confirm that your workspace is using the nested podman devfile and the `nested-podman-scc` is applied on the cluster.

</details>

---

## Part B: Create a Simple Role

Create a minimal Ansible role that installs a package and writes a file:

```bash
mkdir -p ~/molecule-demo && cd ~/molecule-demo
ansible-galaxy role init roles/hello_container
```

Replace the role's `tasks/main.yml`:

```bash
cat > roles/hello_container/tasks/main.yml << 'EOF'
---
- name: Create a test file
  ansible.builtin.copy:
    content: "Hello from Molecule with Podman!\n"
    dest: /tmp/hello_molecule.txt
    mode: "0644"

- name: Verify the file exists
  ansible.builtin.stat:
    path: /tmp/hello_molecule.txt
  register: result

- name: Assert file was created
  ansible.builtin.assert:
    that:
      - result.stat.exists
      - result.stat.size > 0
EOF
```

<details>
<summary>✅ Verification: Role structure is correct</summary>

```bash
ls roles/hello_container/tasks/main.yml
```

**Expected:** File exists with no error.

</details>

---

## Part C: Initialize Molecule with Podman

```bash
cd roles/hello_container
molecule init scenario --driver-name podman
```

Now replace the generated `molecule/default/molecule.yml` to use a UBI container:

```bash
cat > molecule/default/molecule.yml << 'EOF'
---
dependency:
  name: galaxy

driver:
  name: podman

platforms:
  - name: test-instance
    image: registry.access.redhat.com/ubi9/ubi-init:latest
    pre_build_image: true

provisioner:
  name: ansible

verifier:
  name: ansible
EOF
```

Create a verify playbook to confirm the role's outcome:

```bash
cat > molecule/default/verify.yml << 'EOF'
---
- name: Verify hello_container role
  hosts: all
  gather_facts: false
  tasks:
    - name: Read the test file
      ansible.builtin.slurp:
        src: /tmp/hello_molecule.txt
      register: file_content

    - name: Assert file content is correct
      ansible.builtin.assert:
        that:
          - "file_content.content | b64decode == 'Hello from Molecule with Podman!\n'"
EOF
```

<details>
<summary>✅ Verification: Molecule scenario is configured</summary>

```bash
molecule matrix test
```

**Expected:** A list of lifecycle stages (dependency, destroy, syntax, create, converge, idempotency, verify, destroy).

</details>

---

## Part D: Run the Full Test

Execute the complete Molecule lifecycle — this will create a real Podman container, apply the role, verify the result, and destroy the container:

```bash
unset CONTAINER_HOST
molecule test
```

This takes 1–2 minutes. Molecule will:

1. Pull the `ubi-init` image
2. Start a container named `test-instance`
3. Run the role (converge)
4. Run converge again to check idempotency
5. Run the verify playbook
6. Destroy the container

<details>
<summary>✅ Verification: Full test passes</summary>

Look for output ending with:

```
PLAY RECAP *********************************************************************
test-instance              : ok=3    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

INFO     Verifier completed successfully.
INFO     Running default > cleanup
INFO     Running default > destroy
```

All stages should complete without errors. The key line is:
```
INFO     Verifier completed successfully.
```

</details>

---

## Summary

You have successfully:

1. Confirmed nested Podman works inside a Dev Spaces workspace on the cluster
2. Created a simple Ansible role
3. Configured a Molecule scenario with the **podman driver**
4. Executed a full `molecule test` lifecycle with real container isolation

This proves that Molecule can manage real test containers inside OpenShift Dev Spaces when the cluster is configured with the privileged SCC from Setup Step 5.

## Next Module

Proceed to [Module 2: Ansible Development](../../../02-ansible/).

## References

- [Molecule Podman Driver](https://ansible.readthedocs.io/projects/molecule/configuration/#podman)
- [Nested Podman in Dev Spaces](https://github.com/cgruver/devspaces-nested-podman-privileged)
