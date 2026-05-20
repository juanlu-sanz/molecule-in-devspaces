# Step 1: Introduction to Ansible Development Tools

## Objective

Understand what Ansible Development Tools are, verify your development environment is ready, and learn about the Create → Test → Deploy workflow.

## Prerequisites

- Completed Module 1 (Dev Spaces environment configured)
- Running Dev Spaces workspace

---

## Step 1.1: What are Ansible Development Tools?

Ansible Development Tools (often referred to as `ansible-dev-tools` or `adt`) are a curated bundle of command-line tools designed to support the entire Ansible content lifecycle.

These tools cover:

| Category | Tools | Purpose |
|---|---|---|
| **Create** | `ansible-creator`, `ade` | Project scaffolding and environment management |
| **Develop** | `ansible-lint`, VS Code Extension | Linting and IDE support |
| **Test** | `molecule`, `pytest-ansible`, `tox-ansible` | Testing frameworks |
| **Deploy** | `ansible-builder`, `ansible-navigator` | Packaging and EE management |

Instead of assembling and maintaining these tools individually, the bundle provides known-good versions and predictable integration between tools.

<details>
<summary>✅ Verification: Understanding the Tools</summary>

The key benefit of using the bundled tools:
- All contributors use the same tools and compatible versions
- Reduces environment-related issues
- Consistent workflow from development to production

</details>

---

## Step 1.2: Verify Tool Installation

Open a terminal in your Dev Spaces workspace and verify the tools are available.

```bash
adt --version
```

<details>
<summary>✅ Verification: Tools Installed</summary>

**Expected output (versions may vary):**
```
ansible-builder                          3.1.1
ansible-core                             2.17.x
ansible-creator                          24.x.x
ansible-dev-environment                  24.x.x
ansible-dev-tools                        24.x.x
ansible-lint                             24.x.x
ansible-navigator                        24.x.x
molecule                                 24.x.x
pytest-ansible                           24.x.x
tox-ansible                              24.x.x
```

All tools should be listed with version numbers.

</details>

---

## Step 1.3: Explore the VS Code Ansible Extension

The Ansible extension for VS Code provides an enhanced development experience.

1. Click the **Ansible extension icon** (the "A" logo) in the left activity bar
2. You will see the **Ansible Development Tools** section with shortcuts and wizards

The extension provides:

| Feature | Description |
|---|---|
| **Collection scaffolding** | Create new collections with wizard |
| **Playbook scaffolding** | Create playbook projects |
| **Plugin creation** | Add modules, filters, and other plugins |
| **Execution Environment** | Build EE images |
| **Linting integration** | Real-time ansible-lint feedback |
| **Language support** | Syntax highlighting, completion, hover docs |

<details>
<summary>✅ Verification: Extension Active</summary>

1. Open the Ansible extension sidebar (click the "A" icon)
2. You should see sections for:
   - **INITIALIZE** (Collection project, Playbook project, etc.)
   - **ADD** (Collection plugin)
   - **ANSIBLE DEVELOPMENT TOOLS**

If the extension is not visible, reload the VS Code window (`Ctrl+Shift+P` → "Developer: Reload Window").

</details>

---

## Step 1.4: Verify Container Runtime

Molecule and ansible-builder require a container runtime. Verify Podman is available:

```bash
podman --version
```

<details>
<summary>✅ Verification: Podman Available</summary>

**Expected output:**
```
podman version 4.x.x
```

Test that containers can be created:

```bash
podman run --rm registry.access.redhat.com/ubi9/ubi-minimal:latest cat /etc/redhat-release
```

**Expected output:**
```
Red Hat Enterprise Linux release 9.x (Plow)
```

</details>

---

## Step 1.4a: Running `ansible-navigator` in Dev Spaces

Throughout this module, playbook runs use `ansible-navigator` with `--ee false`:

```bash
ansible-navigator run site.yml -i inventory/ --mode stdout --ee false
```

**Why `--ee false`:** Dev Spaces containers run without `/dev/net/tun`, which Podman's network stack (`pasta`) requires to set up a container network interface. Without it, launching a nested EE container fails even though Podman itself is available.

**Can it be enabled?** Yes — a cluster admin can grant `CAP_NET_ADMIN` to the Dev Spaces container, which makes `/dev/net/tun` available and allows EE runs without `--ee false`.

| | `--ee false` | EE enabled (`CAP_NET_ADMIN`) |
|---|---|---|
| Setup required | None | Cluster admin config |
| Runs EE container | No — uses local tools | Yes |
| Matches controller behaviour | Partially | Fully |
| Security risk | None | Elevated privileges on shared cluster |

**In production (automation controller or a local machine with Podman):** drop `--ee false` entirely and pin the EE image in `ansible-navigator.yml`:

```yaml
ansible-navigator:
  execution-environment:
    image: registry.redhat.io/ansible-automation-platform-25/ee-minimal-rhel9:latest
    pull:
      policy: missing
```

---

Create a workspace directory for all the labs in this module:

```bash
mkdir -p /projects/ansible-dev-tools-workspace
cd /projects/ansible-dev-tools-workspace
```

<details>
<summary>✅ Verification: Workspace Created</summary>

```bash
pwd
```

**Expected output:**
```
/projects/ansible-dev-tools-workspace
```

</details>

---

## Step 1.6: The Create → Test → Deploy Workflow

Throughout this module, you will follow a professional content development workflow:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  Step 2 — Playbook Project                                                   │
│  Most basic artifact: inventory + variables + playbook using builtin modules │
└──────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼  (Step 3 adds a Role)
┌──────────────────────────────────────────────────────────────────────────────┐
│  Step 3 — Role                                                               │
│  Package tasks into a reusable role, add it to the playbook                 │
└──────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼  (Step 4 adds the Role to a Collection)
┌──────────────────────────────────────────────────────────────────────────────┐
│  Step 4 — Collection                                                         │
│  Package the role inside a collection, manage deps with ade                 │
└──────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼  (Step 5 adds the Collection to an EE)
┌──────────────────────────────────────────────────────────────────────────────┐
│  Step 5 — Execution Environment                                              │
│  Bundle the collection into a container image with ansible-builder          │
└──────────────────────────────────────────────────────────────────────────────┘
```

| Step | What You'll Learn |
|---|---|
| **Playbook** | Scaffold projects, write inventories, run automation |
| **Role** | Package logic into reusable, testable units |
| **Collection** | Standard distribution format, dependency management with `ade` |
| **EE** | Containerize everything for reproducible execution on AAP |

<details>
<summary>✅ Verification: Workflow Understanding</summary>

This workflow ensures:
- ✓ Content is properly structured
- ✓ Dependencies are managed
- ✓ Code is tested before deployment
- ✓ Artifacts are secure and reproducible

</details>

---

## Summary

You have:

1. Learned what Ansible Development Tools are and how they support the content lifecycle
2. Verified the development tools are installed and ready
3. Explored the VS Code Ansible extension
4. Confirmed container runtime availability
5. Created your workspace directory
6. Understood the Create → Test → Deploy workflow

## Next Step

Proceed to [02-playbook-project](../02-playbook-project/) to scaffold your first playbook project — the most basic Ansible automation artifact.

## References

- [Ansible Development Tools Documentation](https://ansible.readthedocs.io/projects/dev-tools/)
- [Ansible VS Code Extension](https://marketplace.visualstudio.com/items?itemName=redhat.ansible)
- [Getting Started with Ansible Development](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5/html/developing_automation_content/index)
