# Step 4: Creating a Collection

## Objective

Scaffold an Ansible collection using the VS Code extension, set up the development environment with `ade`, add the `role_acmecorp_setup` role from Step 3 into the collection, and update the playbook project to call the role via its Fully Qualified Collection Name (FQCN).

## Prerequisites

- Completed Step 3 (`role_acmecorp_setup` role exists at `/projects/ansible-dev-tools-workspace/roles/role_acmecorp_setup`)
- `acmecorp-playbook` project from Step 2

---

## Step 4.1: Collections — the standard distribution format

A **collection** is the standard way to package and distribute Ansible content. It bundles roles, modules, plugins, playbooks, and tests under a single `namespace.collection_name` identifier.

| Artifact | Lives at |
|---|---|
| Roles | `roles/<role_name>/` |
| Custom modules | `plugins/modules/` |
| Filter plugins | `plugins/filter/` |
| Molecule tests | `extensions/molecule/` |
| Collection identity | `galaxy.yml` |

Using a collection instead of a standalone role gives you:

- **FQCN addressing** — `acme.mycollection.role_acmecorp_setup` is unambiguous regardless of environment
- **Dependency tracking** — `galaxy.yml` declares which other collections are required
- **Single install** — `ade install -e .` installs the collection, all dependencies, and sets up the virtual environment
- **Testability** — Molecule integration (Module 3) works at the collection level
- **Publishability** — collections can be uploaded to Ansible Galaxy or Automation Hub

---

## Step 4.2: Scaffold the collection via the VS Code extension

1. Click the **Ansible extension icon** in the left sidebar (the "A" logo)
2. Under **INITIALIZE**, click **Collection project**
3. Fill in the form:

| Field | Value |
|---|---|
| Namespace | `acme` |
| Collection | `mycollection` |
| Init path | `/projects/ansible-dev-tools-workspace/acme.mycollection` |

4. Check **Install collection from source code (editable mode)**
5. Click **Create**

**CLI equivalent:**

```bash
ansible-creator init collection acme.mycollection \
  /projects/ansible-dev-tools-workspace/acme.mycollection
```

<details>
<summary>✅ Verification: Collection created</summary>

The extension logs should show:

```
------- ansible-creator logs ---------
Note: collection project created at /projects/ansible-dev-tools-workspace/acme.mycollection

------- ansible-dev-environment logs -------
Note: Created virtual environment: /projects/ansible-dev-tools-workspace/acme.mycollection/.venv
Note: Installed collections include: ansible.utils and acme.mycollection
Note: All python requirements are installed.
```

Verify in the terminal:

```bash
ls /projects/ansible-dev-tools-workspace/acme.mycollection/
```

</details>

---

## Step 4.3: Explore the collection structure

Open the collection folder in the VS Code **Explorer** sidebar and inspect the layout:

```
acme.mycollection/
├── .venv/                    ← Virtual environment (created by ade)
├── extensions/
│   └── molecule/             ← Molecule test scenarios (Module 3)
├── meta/
│   └── runtime.yml           ← Minimum Ansible version requirement
├── plugins/
│   ├── filter/               ← Filter plugins
│   ├── modules/              ← Custom Python modules
│   └── ...                   ← Other plugin types
├── tests/
│   ├── integration/          ← Integration tests
│   └── unit/                 ← Unit tests
├── galaxy.yml                ← Collection identity card
├── README.md
└── requirements.txt
```

Click on `galaxy.yml` in the VS Code Editor to activate Ansible extension support and see the scaffolded metadata.

<details>
<summary>✅ Verification: galaxy.yml contains collection identity</summary>

```bash
grep -E "namespace|name|version" /projects/ansible-dev-tools-workspace/acme.mycollection/galaxy.yml
```

**Expected:**
```yaml
namespace: "acme"
name: "mycollection"
version: 1.0.0
```

</details>

---

## Step 4.4: Update galaxy.yml metadata and dependencies

`galaxy.yml` is the **identity card** of the collection. It tells `ade`, `ansible-galaxy`, and Automation Hub everything they need to know: who created it, what it does, which other collections it depends on, and what to exclude from a published tarball.

Open `galaxy.yml` and replace its contents:

```yaml
namespace: "acme"
name: "mycollection"
version: 1.0.0
readme: README.md
authors:
  - Your Name <your.email@example.com>

description: Workshop collection for Ansible Development Tools

license_file: LICENSE

tags:
  - linux
  - tools
  - workshop

dependencies:
  "ansible.utils": "*"
  "ansible.posix": "*"
  "community.general": "*"

build_ignore:
  - .gitignore
  - changelogs/.plugin-cache.yaml
  - .venv
  - collections
  - .tox
```

**`dependencies`** is the machine-readable contract that tells `ade` which collections must be installed. The asterisk (`*`) means latest version; you can also pin with `">=1.5.0"` or `">=1.5.0,<2.0.0"`.

**`build_ignore`** works like `.gitignore` for `ansible-galaxy collection build` — it prevents development artifacts (`.venv`, `collections/`) from bloating the published tarball.

This collection is local and will not be published to Galaxy or Automation Hub, so two rules that require publication metadata can be safely suppressed. Create `.ansible-lint` at the collection root:

```yaml
---
profile: production

skip_list:
  - galaxy[no-repository]   # local workshop collection, not published
  - galaxy[no-changelog]    # no changelog needed for workshop use
```

<details>
<summary>✅ Verification: Metadata updated</summary>

```bash
ansible-lint /projects/ansible-dev-tools-workspace/acme.mycollection/galaxy.yml
```

No errors means the file has a valid namespace, name, version, SPDX licence, and `repository` key.

</details>

---

## Step 4.5: Set up the development environment with ade

`ade` (Ansible Dev Environment) manages the collection's virtual environment and installs all declared dependencies.

First, run `ade install` — this creates the `.venv` and installs all dependencies declared in `galaxy.yml`:

```bash
cd /projects/ansible-dev-tools-workspace/acme.mycollection
ade install -e .
```

Once `ade install` completes and the `.venv` is created, activate it:

```bash
source .venv/bin/activate
```

The `-e .` flag creates an **editable install**: instead of copying the collection into the standard collections path, `ade` creates a symlink. Any change you make to the source is immediately visible — no reinstall cycle needed during development.

<details>
<summary>✅ Verification: All dependencies installed</summary>

```bash
ansible-galaxy collection list
```

**Expected output (versions may differ):**
```
# /projects/ansible-dev-tools-workspace/acme.mycollection/.venv/lib/python3.11/site-packages/ansible_collections
Collection           Version
-------------------- -------
acme.mycollection    1.0.0
ansible.posix        2.x.x
ansible.utils        6.x.x
community.general    12.x.x

# /projects/ansible-dev-tools-workspace/acme.mycollection/.venv/lib64/python3.11/site-packages/ansible/_internal/ansible_collections
Collection           Version
-------------------- -------
ansible._protomatter 2.x.x

# /projects/ansible-dev-tools-workspace/acme.mycollection/.venv/lib64/python3.11/site-packages/ansible_collections
Collection           Version
-------------------- -------
acme.mycollection    1.0.0
ansible.posix        2.x.x
ansible.utils        6.x.x
community.general    12.x.x
```

The three blocks are normal: `lib` and `lib64` are symlinked on most Linux systems so the collections appear twice; `ansible._protomatter` is an internal `ansible-core` collection, not something you installed.

> **Note:** `ade tree` is broken in the bundled image (raises a `KeyError` on transitive dependencies). Use `ansible-galaxy collection list` instead.

Confirm the editable install:

```bash
ls -la /projects/ansible-dev-tools-workspace/acme.mycollection/.venv/lib/python3.11/site-packages/ansible_collections/acme/
```

The `mycollection` entry should be a symlink pointing back to the collection root.

</details>

---

## Step 4.6: Add the role to the collection

The role built in Step 3 now needs to live inside `acme.mycollection` so it is addressable by its FQCN: `acme.mycollection.role_acmecorp_setup`. There are two equivalent ways to do this.

### Option A — VS Code extension (recommended)

1. Click the **Ansible extension icon** in the left sidebar
2. Under **ADD**, click **Role**
3. Fill in the form:

| Field | Value |
|---|---|
| Namespace | `acme` |
| Collection name | `mycollection` |
| Role name | `role_acmecorp_setup` |
| Collection path | `/projects/ansible-dev-tools-workspace/acme.mycollection` |

4. Click **Add**

The extension runs `ansible-creator init collection role` under the hood and scaffolds a fresh role skeleton directly inside `acme.mycollection/roles/role_acmecorp_setup/`. Then copy your content (defaults, tasks, meta) from the standalone role you wrote in Step 3 into the newly created files.

### Option B — CLI

Copy the standalone role directory directly into the collection:

```bash
mkdir -p /projects/ansible-dev-tools-workspace/acme.mycollection/roles
cp -r /projects/ansible-dev-tools-workspace/roles/role_acmecorp_setup \
      /projects/ansible-dev-tools-workspace/acme.mycollection/roles/
```

<details>
<summary>✅ Verification: Role is inside the collection</summary>

```bash
find /projects/ansible-dev-tools-workspace/acme.mycollection/roles -type f | sort
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

</details>

Before linting, fix the scaffolded `tests/test.yml` — the generated play has no `name:`, which violates the `name[play]` best practice. Open the file and replace its contents:

```yaml
# SPDX-License-Identifier: MIT-0
---
- name: Test role_acmecorp_setup
  hosts: localhost
  remote_user: root
  roles:
    - role_acmecorp_setup
```

> The `#SPDX-License-Identifier` comment is scaffolding boilerplate. Best practice is to remove it from test files — it has no legal meaning there and the missing space after `#` triggers `yaml[comments]`. The corrected content above adds the space; alternatively, delete the line entirely.

Then lint the full role:

```bash
cd /projects/ansible-dev-tools-workspace/acme.mycollection
ansible-lint roles/role_acmecorp_setup/
```

> The `WARNING  Another version of...` lines are harmless — `ansible-lint` sees the same collections in both `.ansible/` and `.venv/` and picks the first. No action needed.

---

## Step 4.7: Update the playbook to use the collection via FQCN

Now that the role lives inside `acme.mycollection`, update the playbook project to consume it through the collection rather than via a local `roles_path`.

**Update `acmecorp-playbook/requirements.yml`** to declare the collection dependency:

```yaml
---
collections:
  - name: acme.mycollection
    source: /projects/ansible-dev-tools-workspace/acme.mycollection
    type: dir
```

`type: dir` with a local `source` tells `ansible-galaxy` (and `ade`) to install directly from the source directory. Any change you make to the collection is immediately available in the playbook project — no publish or reinstall cycle during development.

**Install the collection into the playbook project:**

First, deactivate the collection virtualenv — you are now working in the playbook project context, not the collection context:

```bash
deactivate
```

Update `acmecorp-playbook/ansible.cfg` to point to the local collections directory and remove `roles_path` (no longer needed):

```ini
[defaults]
inventory = inventory/
collections_paths = ./collections
```

Then install:

```bash
cd /projects/ansible-dev-tools-workspace/acmecorp-playbook
ansible-galaxy collection install -r requirements.yml -p ./collections
```

**Update `acmecorp-playbook/site.yml`** to use the FQCN:

```yaml
---
- name: Set up development workspace
  hosts: all
  become: false

  roles:
    - role: acme.mycollection.role_acmecorp_setup
```

<details>
<summary>✅ Verification: Playbook uses FQCN and lints cleanly</summary>

```bash
cd /projects/ansible-dev-tools-workspace/acmecorp-playbook
ansible-lint site.yml
```

</details>

Run the playbook to confirm everything still works:

```bash
ansible-navigator run site.yml -i inventory/ --mode stdout --ee false
```

<details>
<summary>✅ Verification: Playbook runs with FQCN role</summary>

**Expected task output:**
```
TASK [acme.mycollection.role_acmecorp_setup : Create workspace base directory] **
ok: [localhost]

TASK [acme.mycollection.role_acmecorp_setup : Create workspace subdirectories] **
ok: [localhost] => (item=projects)
...
```

The FQCN `acme.mycollection.role_acmecorp_setup :` now appears in each task output, confirming the role is resolved through the collection.

</details>

---

## Summary

You have successfully:

1. Scaffolded a collection using the VS Code extension and `ansible-creator`
2. Explored the collection directory structure
3. Updated `galaxy.yml` with metadata, dependencies, and `build_ignore`
4. Set up the development environment with `ade install -e .`
5. Added the `role_acmecorp_setup` role into the collection
6. Updated the playbook project to reference the role via its FQCN

## Collection structure

```
acme.mycollection/
├── roles/
│   └── role_acmecorp_setup/          ← moved from Step 3
│       ├── defaults/main.yml     ← namespaced variables with safe defaults
│       ├── tasks/main.yml        ← create dirs → create config → report
│       └── meta/main.yml
├── galaxy.yml                    ← declares ansible.posix + community.general deps
├── .venv/                        ← ade-managed virtual environment
└── ...
```

## Next Step

Proceed to [05-ee-project](../05-ee-project/) to package the collection into an **Execution Environment** — a container image that bundles `ansible-core`, the collection, its Python dependencies, and required system packages so the playbook runs identically everywhere.

## References

- [ansible-creator Documentation](https://ansible.readthedocs.io/projects/creator/)
- [Ansible Dev Environment Documentation](https://ansible.readthedocs.io/projects/dev-environment/)
- [galaxy.yml Reference](https://docs.ansible.com/ansible/latest/dev_guide/collections_galaxy_meta.html)
- [Collection Dependencies](https://docs.ansible.com/ansible/latest/dev_guide/developing_collections_distributing.html#collection-dependencies)
- [Collection Structure](https://docs.ansible.com/ansible/latest/dev_guide/developing_collections_structure.html)
