# Module 1: OpenShift Dev Spaces

This module gets your team up and running with OpenShift Dev Spaces for Ansible development. It is split into two parts:

1. **Setup** — An administrator installs and configures Dev Spaces on the cluster (a few commands).
2. **Exercises** — Participants work together to create workspace templates and collaborate on Ansible projects.

## Learning Objectives

By the end of this module, you will be able to:

- Install and configure OpenShift Dev Spaces on an OpenShift cluster
- Create a custom workspace template that appears in the Dev Spaces dashboard
- Launch a fully provisioned Ansible development environment in one click
- Collaborate on Ansible code with teammates using a shared repository

## Module Flow

| Section | Duration | Description |
|---|---|---|
| [Setup](./setup/) | ~20 min | Admin installs Dev Spaces and applies base configuration (incl. nested podman) |
| [Exercise 1: Workspace Template](./exercises/01-create-workspace-template/) | ~30 min | Create "My Ansible Workspace" template visible in the dashboard |
| [Exercise 2: Team Collaboration](./exercises/02-team-collaboration/) | ~30 min | Work in groups of 2–4 on a shared Ansible repository |
| [Exercise 3: Molecule with Podman](./exercises/03-molecule-podman-test/) | ~15 min | Run a Molecule test with real containers inside the workspace |

## Prerequisites

- OpenShift 4.17+ cluster with cluster-admin access
- `oc` CLI installed and authenticated to the cluster
- A Git hosting service accessible from the cluster (GitHub, GitLab, Gitea, etc.)
- Groups of 2–4 participants working together

## Quick Reference

```bash
# Check Dev Spaces operator status
oc get csv -n openshift-operators | grep devspaces

# List all workspaces
oc get devworkspaces -A

# Get Dev Spaces dashboard URL
oc get route devspaces -n openshift-devspaces -o jsonpath='https://{.spec.host}{"\n"}'
```

## Next Module

After completing the exercises, proceed to [Module 2: Ansible Development](../02-ansible/).
