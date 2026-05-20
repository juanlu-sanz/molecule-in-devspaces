# Exercise 2: Team Collaboration

## Objective

Work in groups of 2–4 people to simulate a real-world collaboration scenario. Your team will create a shared Git repository, open it in Dev Spaces using the template from Exercise 1, and collaborate on a simple Ansible project — just like you would in your daily work.

## What You Will Learn

- How to open a Git repository directly in Dev Spaces
- How multiple team members work on the same codebase simultaneously
- How Dev Spaces isolates each developer's workspace while sharing code through Git
- Basic Ansible development workflow (edit → lint → commit → push)

---

## Part A: Create a Shared Repository

**One person per team** creates the repository. The rest of the team will be added as collaborators.

### A.1 — Create the repository

Create a new Git repository on your Git hosting service (GitHub, GitLab, Gitea, etc.) with the name:

```
ansible-team-<team-name>
```

For example: `ansible-team-alpha`

Initialize it with a README.

### A.2 — Add team members as collaborators

Add all team members with write access to the repository.

### A.3 — Initialize the project structure

Clone the repository locally or in an existing Dev Spaces workspace, then create the base structure:

```bash
git clone https://<your-git-host>/<org>/ansible-team-<team-name>.git
cd ansible-team-<team-name>
```

Create the basic project files:

```bash
mkdir -p playbooks roles inventory/group_vars
```

Create `ansible.cfg`:

```yaml
# ansible.cfg
[defaults]
inventory = inventory/hosts.yml
roles_path = roles
collections_paths = .ansible/collections
host_key_checking = False
stdout_callback = yaml

[privilege_escalation]
become = True
become_method = sudo
```

Create `inventory/hosts.yml`:

```yaml
---
all:
  hosts:
    localhost:
      ansible_connection: local
```

Create `requirements.yml`:

```yaml
---
collections:
  - name: ansible.posix
  - name: community.general
```

Create `devfile.yaml` at the root of the repository:

```yaml
schemaVersion: 2.2.0
metadata:
  name: ansible-team-<team-name>
  version: 1.0.0
  displayName: "Ansible Team <team-name> Workspace"
  description: "Team workspace for collaborative Ansible development"
attributes:
  controller.devfile.io/storage-type: per-user

components:
  - name: ansible-tools
    container:
      image: registry.redhat.io/devspaces/udi-rhel9:3.27
      memoryLimit: 4Gi
      cpuLimit: 1000m
      cpuRequest: 500m
      mountSources: true
      env:
        - name: ANSIBLE_COLLECTIONS_PATH
          value: ${PROJECT_SOURCE}/.ansible/collections
        - name: ANSIBLE_ROLES_PATH
          value: ${PROJECT_SOURCE}/roles
        - name: ANSIBLE_HOST_KEY_CHECKING
          value: "false"
        - name: ANSIBLE_FORCE_COLOR
          value: "true"
        - name: PY_COLORS
          value: "1"
        - name: MOLECULE_NO_LOG
          value: "false"
        - name: TERM
          value: xterm-256color
      volumeMounts:
        - name: pip-cache
          path: /home/user/.cache/pip
        - name: ansible-data
          path: /home/user/.ansible

  - name: pip-cache
    volume:
      size: 2Gi

  - name: ansible-data
    volume:
      size: 2Gi

commands:
  - id: setup-ansible
    exec:
      component: ansible-tools
      commandLine: |
        pip install --user \
          ansible-dev-tools \
          "molecule-plugins[podman]" \
          pytest-testinfra \
          jmespath \
          netaddr &&
        if [ -f requirements.yml ]; then
          ansible-galaxy collection install -r requirements.yml --force
        fi &&
        echo "✅ Ansible tools installed successfully" &&
        adt --version
      workingDir: ${PROJECT_SOURCE}
      label: "Install Ansible Tools"
  - id: lint
    exec:
      component: ansible-tools
      commandLine: ansible-lint .
      workingDir: ${PROJECT_SOURCE}
      label: "Run Ansible Lint"
  - id: test
    exec:
      component: ansible-tools
      commandLine: molecule test
      workingDir: ${PROJECT_SOURCE}
      label: "Run Molecule Test"

events:
  postStart:
    - setup-ansible
```

> **Note:** When you open this repo from the Dev Spaces dashboard, the dashboard will prompt you to authenticate with your Git provider (GitHub, GitLab, etc.) if the repository is private. This is normal and enables pushing/pulling code from your workspace.

Commit and push:

```bash
git add -A
git commit -m "Initial project structure with devfile"
git push origin main
```

<details>
<summary>✅ Verification: Repository is ready</summary>

All team members should be able to see the repository and it should contain:

```
ansible-team-<team-name>/
├── ansible.cfg
├── devfile.yaml
├── inventory/
│   └── hosts.yml
├── playbooks/
├── requirements.yml
└── roles/
```

</details>

---

## Part B: Open the Repository in Dev Spaces

**Each team member** opens the repository in their own workspace.

### B.1 — Open via the dashboard

1. Go to the Dev Spaces dashboard.
2. In the **"Import from Git"** field, paste your repository URL:
   ```
   https://<your-git-host>/<org>/ansible-team-<team-name>.git
   ```
3. Click **"Create & Open"**.

Dev Spaces will detect the `devfile.yaml` in the repository and use it to configure the workspace automatically.

### B.2 — Wait for the workspace to start

The first start takes ~2–3 minutes. The `postStart` event runs `setup-ansible` which installs `ansible-dev-tools` and all required packages automatically.

<details>
<summary>✅ Verification: Everyone is in</summary>

Each team member opens a terminal in their workspace and runs:

```bash
adt --version
whoami
pwd
```

Everyone should see:
- `adt` version output (confirming the full Ansible Dev Tools suite)
- Their own username
- `/projects/ansible-team-<team-name>` as the working directory

</details>

---

## Part C: Collaborate — Each Member Adds a Playbook

Now simulate a real team workflow. **Each team member** creates a different playbook on their own branch.

### C.1 — Create a feature branch

Each member creates their own branch:

```bash
git checkout -b feature/<your-name>-playbook
```

### C.2 — Write a simple playbook

Each person creates a playbook in `playbooks/` with a unique name. For example:

**Person 1** — `playbooks/system-info.yml`:

```yaml
---
- name: Gather system information
  hosts: all
  gather_facts: true

  tasks:
    - name: Display OS details
      ansible.builtin.debug:
        msg: "{{ ansible_distribution }} {{ ansible_distribution_version }} ({{ ansible_architecture }})"

    - name: Show memory info
      ansible.builtin.debug:
        msg: "Total memory: {{ ansible_memtotal_mb }} MB"
```

**Person 2** — `playbooks/packages.yml`:

```yaml
---
- name: Verify required packages
  hosts: all
  gather_facts: true

  tasks:
    - name: Check Python version
      ansible.builtin.command: python3 --version
      register: python_version
      changed_when: false

    - name: Display Python version
      ansible.builtin.debug:
        msg: "{{ python_version.stdout }}"
```

**Person 3** — `playbooks/users.yml`:

```yaml
---
- name: Display user information
  hosts: all
  gather_facts: true

  tasks:
    - name: Show current user
      ansible.builtin.debug:
        msg: "Running as {{ ansible_user_id }} on {{ ansible_hostname }}"

    - name: List home directory
      ansible.builtin.command: ls -la ~
      register: home_contents
      changed_when: false

    - name: Display home contents
      ansible.builtin.debug:
        msg: "{{ home_contents.stdout_lines[:5] }}"
```

### C.3 — Lint your playbook

```bash
ansible-lint playbooks/<your-playbook>.yml
```

Fix any issues the linter reports.

### C.4 — Commit and push your branch

```bash
git add playbooks/<your-playbook>.yml
git commit -m "Add <description> playbook"
git push origin feature/<your-name>-playbook
```

<details>
<summary>✅ Verification: Branches pushed</summary>

Each team member should have their branch on the remote:

```bash
git branch -r
```

**Expected:** You should see one branch per team member (plus `origin/main`).

</details>

---

## Part D: Merge and Integrate

### D.1 — Merge branches to main

Each member merges their branch (or creates a pull/merge request, depending on your team's preference):

```bash
git checkout main
git pull origin main
git merge feature/<your-name>-playbook
git push origin main
```

If there are merge conflicts, resolve them together — this is part of the learning.

### D.2 — Pull the combined result

Once all branches are merged, everyone pulls the final state:

```bash
git checkout main
git pull origin main
```

### D.3 — Run all playbooks

Verify that everything works together:

```bash
ansible-lint .
ansible-playbook playbooks/system-info.yml
ansible-playbook playbooks/packages.yml
ansible-playbook playbooks/users.yml
```

<details>
<summary>✅ Verification: Team project is complete</summary>

```bash
ls playbooks/
ansible-lint .
```

**Expected:**
- Multiple playbooks in the `playbooks/` directory (one per team member)
- `ansible-lint` passes with no errors

</details>

---

## Discussion Points

Take 5 minutes as a team to discuss:

1. **Isolation** — Each person had their own workspace, but shared the code via Git. How is this different from sharing a single VM or bastion host?
2. **Reproducibility** — Everyone got the same tools because the `devfile.yaml` defined the environment. What happens if someone on the team needs a different Python version?
3. **Speed** — After the first start, subsequent workspace starts are faster due to caching. What would you customize for your real-world team?

---

## Summary

Your team has successfully:

1. Created a shared Git repository with Ansible project structure
2. Added a `devfile.yaml` that auto-provisions the environment
3. Each member opened their own Dev Spaces workspace from the same repo
4. Collaborated using branches (one playbook per person)
5. Merged all contributions and verified the combined result

This mirrors a real-world Ansible team workflow — isolated environments, shared code, consistent tooling.

## Next Module

Proceed to [Module 2: Ansible Development](../../../02-ansible/) to learn about collections, roles, and advanced playbook patterns.
