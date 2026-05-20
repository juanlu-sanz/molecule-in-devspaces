# Module 2: Ansible Development Tools

This module provides a comprehensive, hands-on learning experience for understanding the developer flow for automation content using the Ansible Development Tools suite.

## Learning Objectives

By the end of this module, you will be able to:

- Use the **Ansible extension for VS Code** features for a rich editing experience
- Learn all the creation tools, including `ansible-creator` to scaffold new collections and content, `adt` (Ansible Development Tools) for unified command-line tools management, and `ade` (Ansible Development Environment) for managing collection dependencies and virtual environments
- Test your content with `ansible-lint` for checking playbooks against recommended practices, and `molecule` for testing your collections in isolated, reproducible environments
- Deploy to production by packaging your collection content with `ansible-galaxy` and `ansible-builder`

## Module Structure

The module follows a **basic → advanced** progression, building each artifact on top of the previous one:

| Step | Duration | Description |
|---|---|---|
| [01-introduction](./01-introduction/) | 15 min | Introduction to Ansible Development Tools |
| [02-playbook-project](./02-playbook-project/) | 20 min | Creating a standalone playbook project |
| [03-role-project](./03-role-project/) | 25 min | Creating a reusable role and adding it to the playbook |
| [04-collection](./04-collection/) | 40 min | Creating a collection and adding the role to it |
| [05-ee-project](./05-ee-project/) | 35 min | Creating an Execution Environment with the collection |

### Progressive artifact chain

```
Playbook project  ←  Role added in Step 3
      ↓
    Role  ←  Added to Collection in Step 4
      ↓
  Collection  ←  Added to EE in Step 5
      ↓
     EE
```

## Prerequisites

- Completed Module 1 (Dev Spaces environment configured)
- Running Dev Spaces workspace with Ansible Development Tools
- Basic understanding of Linux command line operations
- Familiarity with VS Code
- Basic Ansible and YAML syntax understanding

## Quick Reference

### Ansible Development Tools Components

| Tool | Purpose |
|---|---|
| `ansible-creator` | Scaffold collections, playbooks, and plugins |
| `ade` | Manage development environments and dependencies |
| `adt` | Unified command-line tools management |
| `ansible-lint` | Check playbooks against best practices |
| `molecule` | Test collections in isolated environments |

| `ansible-builder` | Create Execution Environments |
| `ansible-navigator` | Run and test Execution Environments |

### Key Commands

```bash
# Check tool versions
adt --version

# Create a collection
ansible-creator init collection acme.mycollection /path/to/collection

# Manage dependencies
ade install -e .

# Run linter
ansible-lint .

# Run Molecule tests
molecule test -s <scenario>

# Build Execution Environment
ansible-builder build -t my-ee:latest
```

## References

- [Ansible Development Tools Documentation](https://ansible.readthedocs.io/projects/dev-tools/)
- [Ansible Creator Documentation](https://ansible.readthedocs.io/projects/creator/)
- [Ansible Dev Environment Documentation](https://ansible.readthedocs.io/projects/dev-environment/)
- [Molecule Documentation](https://ansible.readthedocs.io/projects/molecule/)
- [Ansible Builder Documentation](https://ansible.readthedocs.io/projects/builder/)
