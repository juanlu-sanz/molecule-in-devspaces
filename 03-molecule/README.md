# Module 3: Testing with Molecule

This module introduces Molecule as the standard testing framework for Ansible collections. You will test the objects built in Module 2 — specifically the `role_acmecorp_setup` role inside `acme.mycollection` — using the **podman driver** to create real test containers inside the Dev Spaces workspace.

## Learning Objectives

By the end of this module, you will be able to:

- Explain what Molecule does and why collections use it
- Describe the Molecule lifecycle stages
- Configure Molecule to use the podman driver for full container isolation
- Run the full lifecycle with `molecule test`
- Use granular commands (`create`, `converge`, `verify`, `login`, `destroy`) for faster development iteration
- Write an integration test for the `role_acmecorp_setup` role from Module 2
- Create a new collection with a RHEL System Roles dependency and test its validation logic

## Module Structure

| Step | Duration | Description |
|---|---|---|
| [01-introduction](./01-introduction/) | 15 min | What Molecule is, the podman driver, and collection structure |
| [02-running-tests](./02-running-tests/) | 20 min | Run the scaffolded `integration_hello_world` scenario with podman |
| [03-workspace-role](./03-workspace-role/) | 25 min | Write and run a Molecule test for `role_acmecorp_setup` |
| [04-bonus](./04-bonus/) | 30 min | **Bonus** — Test a filesystem setup role wrapping `redhat.rhel_system_roles.storage` |

## Prerequisites

- Completed Module 2 (collection `acme.mycollection` with `role_acmecorp_setup` role)
- Workspace running with the nested podman image ([Module 1 Setup Step 5](../01-devspaces/setup/))
- Podman working inside the workspace (`podman run --rm ubi-minimal echo test`)

## Quick Reference

### Molecule lifecycle (full)

```
dependency → destroy → syntax → create → converge → idempotency → verify → cleanup → destroy
```

### Key commands

```bash
# Run the full lifecycle for a scenario
molecule test -s <scenario_name>

# Run individual stages for faster iteration
molecule create -s <scenario_name>     # start the container
molecule converge -s <scenario_name>   # apply the playbook
molecule verify -s <scenario_name>     # run assertions
molecule login -s <scenario_name>      # shell into the container
molecule destroy -s <scenario_name>    # stop and remove the container

# Inspect running containers
molecule list -s <scenario_name>
```

### Working directory

All Molecule commands run from the `extensions/` directory inside the collection:

```bash
cd /projects/ansible-dev-tools-workspace/acme.mycollection/extensions
```

## References

- [Molecule Documentation](https://ansible.readthedocs.io/projects/molecule/)
- [Molecule Configuration Reference](https://ansible.readthedocs.io/projects/molecule/configuration/)
- [Molecule Podman Driver](https://ansible.readthedocs.io/projects/molecule-plugins/podman/)
- [Ansible Testing Guide](https://docs.ansible.com/ansible/latest/dev_guide/testing.html)
