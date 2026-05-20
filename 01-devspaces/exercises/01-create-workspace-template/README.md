# Exercise 1: Create "My Ansible Workspace" in the Dashboard

## Objective

Register a **clickable workspace sample** in the Dev Spaces dashboard called **"My Ansible Workspace"**. When any team member clicks it, Dev Spaces will spin up a fully provisioned Ansible development environment — with a specific Ansible version and all required tools — in under 3 minutes.

By the end of this exercise your team will have a one-click tile in the Dev Spaces dashboard that anyone can use to start developing Ansible immediately.

## What You Will Learn

- What a `devfile.yaml` is and how it defines a workspace environment
- How to register a workspace as a clickable sample in the Dev Spaces dashboard
- How Dev Spaces auto-provisions tools when a workspace starts

---

## Part A: Understand the Devfile

A **devfile** is a YAML file that lives at the root of a Git repository. It tells Dev Spaces what container image to use, what tools to install, and how to configure the environment.

Look at the `devfile.yaml` below. This is what Dev Spaces will use to create workspaces when someone clicks the sample tile.

Create a file called `devfile.yaml`:

```yaml
schemaVersion: 2.2.0
metadata:
  name: my-ansible-workspace
  version: 1.0.0
  displayName: My Ansible Workspace
  description: Pre-configured Ansible development environment with ansible-dev-tools, molecule, and linting tools
  language: yaml
  projectType: ansible
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

Next, create a `.vscode/` directory with two files to pre-configure extensions and editor settings. Dev Spaces picks these up automatically from the repository.

Create `.vscode/extensions.json`:

```json
{
  "recommendations": [
    "redhat.ansible",
    "redhat.vscode-yaml",
    "dracula-theme.theme-dracula"
  ]
}
```

Create `.vscode/settings.json`:

```json
{
  "workbench.colorTheme": "Solarized Dark",
  "ansible.python.interpreterPath": "/usr/bin/python3",
  "ansible.validation.lint.enabled": true,
  "ansible.validation.enabled": true,
  "files.associations": {
    "*.yml": "ansible",
    "*.yaml": "ansible"
  },
  "editor.tabSize": 2,
  "editor.detectIndentation": false
}
```

These files ensure that when the workspace starts, the editor will:
- Auto-install the **Red Hat Ansible** extension (syntax highlighting, auto-completion, linting)
- Auto-install the **Red Hat YAML** extension (schema validation)
- Apply the **Solarized Dark** theme (with Dracula available as a recommended extension)
- Associate all `.yml`/`.yaml` files with the Ansible language mode

> **Note:** These extensions are resolved from the Open VSX registry configured in your Dev Spaces instance. In disconnected environments, the administrator must ensure these extensions are available in the local Open VSX mirror. See [Configure the Open VSX registry URL](https://eclipse.dev/che/docs/stable/administration-guide/configuring-the-open-vsx-registry-url/) for details.

**Key points to discuss with your team:**

| Section | What it does |
|---|---|
| `components[].container.image` | The base container image (Red Hat UDI with Python, Git, etc.) |
| `components[].container.env` | Environment variables for Ansible paths, colors, and Molecule logging |
| `commands[].exec.commandLine` | Tools installed automatically when the workspace starts (`ansible-dev-tools` bundles ansible-core, molecule, ansible-lint, ansible-creator, ansible-builder, ansible-navigator, and more) |
| `events.postStart` | Triggers the setup command on every workspace start |
| `volumes` | Caches pip packages and Ansible collections/roles so re-starts are faster |
| `.vscode/extensions.json` | Auto-installs recommended VS Code extensions (Ansible, YAML, Dracula theme) |
| `.vscode/settings.json` | Pre-configures the editor: Solarized Dark theme, Ansible linting, YAML file associations |

---

## Part B: Push the Devfile to a Git Repository

For the sample tile to work, the `devfile.yaml` must live in a Git repository that Dev Spaces can reach.

**Option 1 — Use the workshop repository** (simplest):

The instructor provides the Git URL of this workshop repository, which already contains a devfile:
```
https://github.com/<org>/workshop-devspaces-ansible-molecule.git
```

**Option 2 — Create a minimal repository** (for teams who want full control):

One team member creates a new repository, adds the `devfile.yaml` from Part A, and pushes it:
```bash
mkdir my-ansible-workspace && cd my-ansible-workspace
git init
# (copy the devfile.yaml from Part A into this directory)
git add devfile.yaml
git commit -m "Add Ansible workspace devfile"
git remote add origin https://<your-git-host>/<org>/my-ansible-workspace.git
git push -u origin main
```

Take note of the **HTTPS Git URL** — you'll need it in the next step.

> **Note:** The repository must be accessible from the cluster. If it's private, the Dev Spaces dashboard will prompt participants to authenticate with your Git provider (GitHub OAuth, GitLab token, etc.) when they click the tile.

<details>
<summary>✅ Verification: Repository has a devfile</summary>

```bash
# The repo should be accessible and contain devfile.yaml at the root
git ls-remote <your-repo-url> HEAD
```

This should return a commit hash without errors.

</details>

---

## Part C: Register the Sample in the Dashboard

Now create a ConfigMap that tells the Dev Spaces dashboard to show your workspace as a clickable tile.

Create a file called `my-ansible-workspace-sample.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-ansible-workspace-sample
  namespace: openshift-devspaces
  labels:
    app.kubernetes.io/part-of: che.eclipse.org
    app.kubernetes.io/component: getting-started-samples
data:
  .sample0: |
    {
      "displayName": "My Ansible Workspace",
      "description": "Pre-configured Ansible environment with ansible-dev-tools, molecule, and linting. Click to start developing!",
      "tags": ["Ansible", "YAML", "Automation"],
      "url": "https://github.com/<org>/<repo>.git",
      "icon": {
        "base64data": "PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCAyNCAyNCI+PHBhdGggZmlsbD0iI0VFMDAwMCIgZD0iTTEyIDJDNi40OCAyIDIgNi40OCAyIDEyczQuNDggMTAgMTAgMTAgMTAtNC40OCAxMC0xMFMxNy41MiAyIDEyIDJ6bTAgMThjLTQuNDEgMC04LTMuNTktOC04czMuNTktOCA4LTggOCAzLjU5IDggOC0zLjU5IDgtOCA4em0tMi0zLjVsNi0zLjUtNi0zLjV2N3oiLz48L3N2Zz4=",
        "mediatype": "image/svg+xml"
      }
    }
```

**Replace** `https://github.com/<org>/<repo>.git` with your actual repository URL from Part B.

Apply it:

```bash
oc apply -f my-ansible-workspace-sample.yaml
```

<details>
<summary>✅ Verification: Sample is registered</summary>

```bash
oc get configmap -n openshift-devspaces -l app.kubernetes.io/component=getting-started-samples
```

**Expected:**
```
NAME                          DATA   AGE
my-ansible-workspace-sample   1      <age>
```

</details>

---

## Part D: Click It!

1. Open the Dev Spaces dashboard:

```bash
oc get route devspaces -n openshift-devspaces -o jsonpath='https://{.spec.host}{"\n"}'
```

2. On the **"Create Workspace"** page, you should see **"My Ansible Workspace"** as a sample tile with the Ansible description.

3. Click it. Dev Spaces will:
   - Clone the repository
   - Start a container with the UDI image
   - Automatically run the `setup-ansible` command (installs `ansible-dev-tools`, molecule with podman driver, and testing utilities)

4. Wait ~2–3 minutes for the workspace to start and the tools to install.

5. Open a terminal in the workspace and verify:

```bash
adt --version
molecule --version
ansible-lint --version
```

<details>
<summary>✅ Verification: Workspace tools are installed</summary>

All three commands should return version information:

- `adt` — confirms the full Ansible Dev Tools suite is installed (includes ansible-core, ansible-creator, ansible-builder, ansible-navigator, ansible-lint, molecule, and more)
- `molecule 24.x.x` — testing framework
- `ansible-lint 24.x.x` — linting tool

If the tools aren't ready yet, wait a minute — the `postStart` event runs automatically in the background.

</details>

---

## Discussion

Take 5 minutes to discuss with your team:

1. **One-click onboarding** — New team members can start contributing from day one. No local setup needed.
2. **Version pinning** — The devfile installs `ansible-dev-tools` (which bundles a compatible ansible-core). What happens when the team wants to upgrade or pin a specific version?
3. **Customization** — Each team could have their own sample tile. What would you add to yours?

---

## Summary

You have:

1. Learned what a `devfile.yaml` is and how it defines a workspace
2. Registered a clickable sample in the Dev Spaces dashboard
3. Launched a workspace with `ansible-dev-tools` (ansible-core, molecule, ansible-lint, and more) pre-installed
4. Got Red Hat Ansible and YAML extensions auto-installed with the Solarized Dark theme applied
5. Experienced one-click workspace creation for your team

## Next Exercise

Proceed to [Exercise 2: Team Collaboration](../02-team-collaboration/) to create your own team repository and collaborate on Ansible code.
