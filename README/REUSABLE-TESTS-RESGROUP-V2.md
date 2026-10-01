# Greengage Reusable Resource Group v2 Tests Workflow

This workflow runs the **resource group v2** isolation test suite
(`installcheck-resgroup-v2`) for the Greengage project. It is designed to be
called from a parent CI pipeline.

## Actual version

- `greengagedb/greengage-ci/.github/workflows/greengage-reusable-tests-resgroup-v2.yml@v57`

## Purpose

The workflow runs `src/test/isolation2` resource group v2 tests against a demo
cluster built from the SHA-tagged Greengage image. Resource group v2 (cgroup v2)
is a Greengage 7.x feature, so this workflow is 7.x only. The suite is run for
both the `postgres` and `orca` optimizers.

## How it differs from `tests-resgroup` (cgroup v1)

| | `tests-resgroup` (v1) | `tests-resgroup-v2` (this workflow) |
|---|---|---|
| cgroup mode | v1 | v2 |
| Execution | QEMU VM booted with `systemd.unified_cgroup_hierarchy=0` | Directly on the runner |
| Make target | `installcheck-resgroup` (`src/test/regress`) | `installcheck-resgroup-v2` (`src/test/isolation2`) |

GitHub-hosted `ubuntu-24.04` runners already boot with cgroup v2, so this
workflow does **not** need a VM. The job runs `concourse/scripts/ic_gpdb_resgroup_v2.bash`
directly in a privileged container that shares the host cgroup namespace
(`--cgroupns=host`) and the host cgroup v2 filesystem
(`-v /sys/fs/cgroup:/sys/fs/cgroup:rw`). That script creates `/sys/fs/cgroup/gpdb`,
enables the `cpuset io cpu memory` controllers and runs the tests.

`gpdemo-datadirs` and the isolation2 `testtablespace` are bind-mounted from the
runner's `/mnt` disk (a real block device) because IO_LIMIT tests need real
block devices and the container script rejects Docker overlay / tmpfs paths.

## Usage

### Inputs

| Name                | Description                                       | Required | Type    | Default |
|---------------------|--------------------------------------------------|----------|---------|---------|
| `version`           | Greengage version (`7`; 7.x only)                | Yes      | String  | -       |
| `target_os`         | Target operating system (e.g., `ubuntu`)         | Yes      | String  | -       |
| `target_os_version` | Target OS version (e.g., ``, `24.04`)            | No       | String  | `''`    |

### Secrets

| Name         | Description                  | Required |
|--------------|------------------------------|----------|
| `ghcr_token` | GitHub token for GHCR access | Yes      |

### Requirements

- **Permissions**: `contents: read`, `packages: read`, `actions: write`.
- **Docker Image**: a SHA-tagged image must exist in GHCR matching
  `ghcr.io/<repo>/ggdb<version>_<target_os><target_os_version>:<sha>`.
- **Runner**: a GitHub-hosted `ubuntu-24.04` runner (cgroup v2, privileged
  Docker, KVM not required).
- **Repository scripts**: `concourse/scripts/ic_gpdb_resgroup_v2.bash` must be
  present in the image.

### Environment Variables

- `STATEMENT_MEM`: `125MB`
- `TEST_OS`: target operating system
- `OPTIMIZER`: `on` for ORCA, `off` for Postgres

### Single Job Example

```yaml
jobs:
  resgroup-v2-tests:
    permissions:
      contents: read
      packages: read
      actions: write
    uses: greengagedb/greengage-ci/.github/workflows/greengage-reusable-tests-resgroup-v2.yml@v57
    with:
      version: 7
      target_os: ubuntu
    secrets:
      ghcr_token: ${{ secrets.GITHUB_TOKEN }}
```

## Artifacts

- **Artifact Name**: `resgroup_v2_ggdb<version>_<target_os><target_os_version>_<optimizer>`
- **Content**: `gpAdminLogs`, isolation2 `results`/`resgroup` output,
  `regression.diffs`, demo cluster logs
- **Retention**: 7 days
