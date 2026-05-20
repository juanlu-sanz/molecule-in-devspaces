# Workshop: Ansible Development with Molecule on OpenShift Dev Spaces

A full-day, hands-on workshop designed for teams interested in running Ansible playbooks and testing with Molecule inside OpenShift Dev Spaces.

## Workshop Overview

This workshop guides you through three progressive modules:

1. **OpenShift Dev Spaces** — Quick setup + hands-on exercises (workspace templates, team collaboration)
2. **Ansible Development** — Collections, roles, and playbook development
3. **Molecule Testing** — Automated testing of Ansible roles within Dev Spaces

Each module builds on the previous one, culminating in a complete Ansible development environment running entirely in OpenShift Dev Spaces. Participants work in groups of 2–4 people throughout.

## Environment

| Component | Version |
|---|---|
| OpenShift Container Platform | 4.17+ |
| OpenShift Dev Spaces | 3.19+ |
| Ansible Core | 2.17+ |
| Molecule | 24.x |
| Python | 3.11+ |

## Prerequisites

- Access to an OpenShift 4.17+ cluster with cluster-admin privileges
- `oc` CLI installed and configured
- Basic familiarity with containers and Kubernetes concepts
- Basic Ansible knowledge (helpful but not required)

## Workshop Structure

| Module | Duration | Description |
|---|---|---|
| [Module 1: OpenShift Dev Spaces](./01-devspaces/) | ~1.25 hours | Quick setup + exercises: workspace templates and team collaboration |
| [Module 2: Ansible Development](./02-ansible/) | ~2 hours | Create collections, develop roles, and write playbooks |
| [Module 3: Molecule Testing](./03-molecule/) | ~2.5 hours | Test Ansible roles using Molecule in Dev Spaces |

## Quick Start

1. Start with [Module 1](./01-devspaces/) to set up OpenShift Dev Spaces on your cluster
2. Progress to [Module 2](./02-ansible/) to learn Ansible development patterns
3. Complete [Module 3](./03-molecule/) to integrate automated testing

## Repository Map

| Directory | Purpose |
|---|---|
| `01-devspaces/` | Dev Spaces setup (admin) and hands-on exercises (participants) |
| `02-ansible/` | Ansible collections, roles, and playbook development |
| `03-molecule/` | Molecule testing scenarios and integration examples |
| `resources/` | Shared resources, devfiles, and container images |

## How to Use This Workshop

Each step in every module includes:

- **Objective** — What you will accomplish
- **Instructions** — Step-by-step commands and explanations
- **Expected Output** — What success looks like
- **Verification** — Expandable section to confirm the step completed correctly

### Verification Sections

Throughout this workshop, you will find verification sections formatted as expandable details:

<details>
<summary>✅ Verification: Example Check</summary>

Run this command to verify the step completed successfully:

```bash
oc get pods -n example-namespace
```

**Expected output:**
```
NAME                    READY   STATUS    RESTARTS   AGE
example-pod-abc123      1/1     Running   0          2m
```

If you see this output, proceed to the next step.

</details>

## References

- [OpenShift Dev Spaces Documentation](https://docs.redhat.com/en/documentation/red_hat_openshift_dev_spaces/3.19)
- [OpenShift Container Platform Documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17)
- [Red Hat Ansible Automation Platform Documentation](https://docs.redhat.com/en/documentation/red_hat_ansible_automation_platform/2.5)
- [Ansible Documentation](https://docs.ansible.com/ansible/latest/)
- [Molecule Documentation](https://ansible.readthedocs.io/projects/molecule/)

## License

This workshop content is provided for educational purposes.
