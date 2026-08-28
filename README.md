# Greengage CI Workflows

This repository contains reusable GitHub Actions workflows for building,
testing, packaging, and publishing **Greengage Database (GGDB)** Docker images.
These workflows are designed to be called from a parent CI pipeline in the
`GreengageDB/greengage` repository.

## ⚠️ Important Notice

Whenever the list of **NAMES of required jobs** in the workflow (including any
**reusable workflows**) is **added, removed, or renamed**, you must contact a
repository administrator to update the **Branch Protection Rules** accordingly.
Without this, new, deleted, or renamed jobs will not be recognized as required
when checking Pull Requests.

## Purpose

The repository provides the following reusable workflows:

- `build`: Builds GGDB Docker images and pushes to GHCR with commit SHA tag.
- `tests-behave`: Executes Behave test suites, generates artifacts and report.
- `tests-orca`: Runs ORCA linter and unit tests, produces test artifacts.
- `tests-regression`: Performs regression tests with optimizer settings.
- `tests-resgroup`: Conducts resource groups (cgroup v1) tests using QEMU VM.
- `tests-resgroup-v2`: Conducts resource group v2 (cgroup v2) isolation tests
  directly on the runner (version 7.x only).
- `tests-jit`: Performs JIT tests with optimizer settings (version 7.x only).
- `tests-pg_upgrade`: Verifies the 6.x → 7.x `pg_upgrade` path against the
  built target image (Ubuntu only).
- `package`: Builds Debian packages and tests deployment (version 6.x only).
- `upload`: Retags and uploads GGDB images to GHCR and DockerHub.
- `cleanup`: **(WIP)** Deletes branch images from GHCR. Not in use.

### Algorithm Overview

1. **`build`**:

   - Builds a GGDB Docker image using `ci/Dockerfile.<target_os>`.
   - Tags the image with the full commit SHA.
   - Pushes the image to GHCR.
   - Uses inputs: `version`, `target_os`, `target_os_version`.

2. **`tests-behave`**:

   - Runs Behave tests for features in a Docker container.
   - Generates artifacts and an aggregated Allure report.
   - Uses a matrix strategy to test multiple features.
   - Uses inputs: `version`, `target_os`, `target_os_version`.

3. **`tests-orca`**:

   - Executes ORCA linter and unit tests in a Docker container.
   - Builds a linter image using `ci/Dockerfile.linter`.
   - Uploads artifacts.
   - Uses inputs: `version`, `target_os`, `target_os_version`.

4. **`tests-regression`**:

   - Runs regression tests with optimizer settings (`orca`, `postgres`).
   - Uses a custom GitHub Action for test execution.
   - Generates log artifacts stored in a volume.
   - Uses a matrix strategy to test both optimizer settings.
   - Uses inputs: `version`, `target_os`, `target_os_version`.

5. **`tests-resgroup`**:

   - Executes resource groups (cgroup v1) tests using QEMU VM with cloud-init.
   - Generates log artifacts.
   - Uses inputs: `version`, `target_os`, `target_os_version`.

6. **`tests-resgroup-v2`**:

   - Executes resource group v2 (cgroup v2) isolation tests
     (`installcheck-resgroup-v2`) via `ci/scripts/run_resgroup_v2_test.bash`.
   - Runs directly on the `ubuntu-24.04` runner (which already provides
     cgroup v2), in a privileged container sharing the host cgroup namespace
     and cgroup v2 filesystem - no QEMU VM.
   - Uses a matrix strategy for the `orca` / `postgres` optimizers (v7 only).
   - Generates log artifacts.
   - Uses inputs: `version`, `target_os`, `target_os_version`.

7. **`tests-jit`**:

   - Runs JIT tests with optimizer settings (`orca`, `postgres`).
   - Generates log artifacts stored in a volume.
   - Uses a matrix strategy to test both optimizer settings.
   - Uses inputs: `version`, `target_os`, `target_os_version`.

8. **`tests-pg_upgrade`**:

   - Restores the built target image, builds a combined 6.x/7.x test
     image on top of it, and runs `pg_upgrade` to verify the upgrade
     path.
   - Uploads regression diffs as artifacts, always.
   - Ubuntu only; `target_os`/`target_os_version` are accepted for
     matrix consistency with other workflows but not otherwise varied.
   - Uses inputs: `version`, `target_os`, `target_os_version`.

9. **`package`**:

   - Builds Debian packages inside the Docker container.
   - Optionally tests deployment in Docker.
   - Uploads and caches Debian artifacts.
   - Uses inputs: `version`, `target_os`, `target_os_version`.

10. **`upload`**:

    - Restores and loads the SHA-tagged image from cache.
    - Retags based on the event:
      - For tagged push: Uses git tag (e.g., `6.28.0`).
      - For branch push: Uses `latest` (GHCR) or `testing` (DockerHub).
    - Pushes to GHCR and optionally to DockerHub.
    - Uses inputs: `version`, `target_os`, `target_os_version`.

11. **`cleanup`** **(WIP — not in use)**:

    - Deletes branch-related images from GHCR on branch deletion.
    - Uses crane and GitHub API for tag removal.
    - Uses inputs: `version`, `target_os`, `target_os_version`.

## Usage

To integrate these workflows into your pipeline:

1. Add jobs in your parent workflow that call the reusable workflows from this
   repository.
2. Provide the required inputs as described below.
3. Ensure the necessary permissions and secrets are configured.

### Inputs

| Name | Description | Required | Type |
|------|-------------|----------|------|
| `version` | Greengage version (`6` or `7`) | Yes | String |
| `target_os` | Target OS (e.g., `ubuntu`) | Yes | String |
| `target_os_version` | Target OS version (`22.04`, `24.04`) | No | String |

### Secrets

| Name | Description | Required |
|------|-------------|----------|
| `ghcr_token` | GitHub token for GHCR access | Yes |
| `DOCKERHUB_USERNAME` | DockerHub username (for upload) | No |
| `DOCKERHUB_TOKEN` | DockerHub token (for upload) | No |

### Requirements

- **Permissions**:
  - `packages: read` — for pulling images from GHCR in test workflows.
  - `packages: write` — for pushing images to GHCR in `build`, `upload`.
  - `actions: write` — for uploading artifacts in test workflows.
  - `contents: write` — for uploading packages to release (if enabled).

- **Secrets**: Provide a `GITHUB_TOKEN` with sufficient permissions as the
  `ghcr_token` secret.

- **Docker Image**: For test and upload workflows, ensure a Docker image exists
  in GHCR matching the format
  `ghcr.io/<repo>/ggdb<version>_<target_os><target_os_version>:<full-sha>`.

- **Repository Access**: Workflows check out the repository specified in
  `github.repository`.

- **Artifacts**: Test workflows upload artifacts (e.g., `allure-results`,
  `logs_cdw`, `regression_logs`, `jit_*`, `resgroup_*`).

### Important Notes on `target_os_version`

> **⚠️ BACKWARD COMPATIBILITY WARNING**
>
> For `ubuntu`, specifying `target_os_version: "22.04"` explicitly is **not
> recommended** and may break backward compatibility with previous CI versions.
>
> **Reason**: In earlier CI versions, Ubuntu version was not versioned — it was
> hardcoded as the only possible option. The version did not appear anywhere in
> the configuration.
>
> **Recommended approach**:
> - For `ubuntu`, **omit** `target_os_version` (leave it empty) to use the
>   default behavior.
> - Specify `target_os_version: "24.04"` only when you explicitly need Ubuntu
>   24.04.
>
> **Example**:
> ```yaml
> # Correct for default Ubuntu (recommended)
> - target_os: ubuntu
>
> # Correct for explicit Ubuntu 24.04
> - target_os: ubuntu
>   target_os_version: "24.04"
>
> # NOT recommended (breaks backward compatibility)
> - target_os: ubuntu
>   target_os_version: "22.04"
> ```

### Examples

Real-world examples are available in the [`examples/`](examples/) directory:

| File | Description |
|------|-------------|
| [`greengage-ci-6x.yml`](examples/greengage-ci-6x.yml) | CI for 6.x |
| [`greengage-ci-7x.yml`](examples/greengage-ci-7x.yml) | CI for 7.x |
| [`greengage-release.yml`](examples/greengage-release.yml) | Release upload |
| [`greengage-sql-dump.yml`](examples/greengage-sql-dump.yml) | SQL dump |

#### Key Differences Between 6.x and 7.x

- **Version 6.x**:
  - Builds for both default Ubuntu and Ubuntu 24.04.
  - Does **not** include `jit-tests` (JIT not supported in v6).
  - `package` job includes `test_docker` for deployment testing.

- **Version 7.x**:
  - Builds for default Ubuntu only.
  - Includes `jit-tests` job.
  - Simpler matrix configuration.

#### Common Patterns

Both versions share these patterns:

- **Concurrency control**: Cancels previous runs on new push to same PR/branch.
- **Conditional test execution**: Tests run only on `pull_request`, not on push.
- **Conditional upload**: Upload runs only on `push` (to main or tags).
- **Matrix strategy**: `fail-fast` varies by job type.

### Notes

- Workflows assume the default branch is checked out unless specified otherwise.
- Docker images are tagged with the full commit SHA and target OS version
  (e.g., `ghcr.io/<owner>/<repo>/ggdb6_<target_os><target_os_version>:<sha>`).
- The `upload` workflow pushes to GHCR and optionally to DockerHub (if
  credentials are provided).
- The `resgroup-tests` workflow uses QEMU VM, requiring significant resources
  (4 CPUs, 8GB memory).
- The `resgroup-v2-tests` workflow runs directly on the `ubuntu-24.04` runner
  (cgroup v2, privileged Docker); it does not use a QEMU VM.
- The `regression-tests` workflow uses custom `sysctl` settings.
- Ensure the CI directory structure and required scripts are present in the
  repository (e.g., `run_behave_tests.bash`, `run_resgroup_test.bash`,
  `run_resgroup_v2_test.bash`, `ic_gpdb.bash`, `unit_tests_gporca.bash`).
- Artifacts are uploaded with names like
  `<job>_ggdb<version>_<target_os><target_os_version>_<suffix>`.
- The `cleanup` workflow should be triggered on branch deletion to remove
  related Docker images from GHCR.

## Additional Documentation

Detailed README files for each process are available in the
[README](https://github.com/greengagedb/greengage-ci/blob/main/README/)
directory of this repository:

- Build process:
  [REUSABLE-BUILD.md](README/REUSABLE-BUILD.md)
- Package process:
  [REUSABLE-PACKAGE.md](README/REUSABLE-PACKAGE.md)
- Behave tests:
  [REUSABLE-TESTS-BEHAVE.md](README/REUSABLE-TESTS-BEHAVE.md)
- Orca tests:
  [REUSABLE-TESTS-ORCA.md](README/REUSABLE-TESTS-ORCA.md)
- Regression tests:
  [REUSABLE-TESTS-REGRESSION.md](README/REUSABLE-TESTS-REGRESSION.md)
- Resource group tests:
  [REUSABLE-TESTS-RESGROUP.md](README/REUSABLE-TESTS-RESGROUP.md)
- Resource group v2 tests:
  [REUSABLE-TESTS-RESGROUP-V2.md](README/REUSABLE-TESTS-RESGROUP-V2.md)
- pg_upgrade tests:
  [REUSABLE-TESTS-PG_UPGRADE.md](README/REUSABLE-TESTS-PG_UPGRADE.md)
- Upload process:
  [REUSABLE-UPLOAD.md](README/REUSABLE-UPLOAD.md)

## Contributing

For issues or contributions, please open a pull request or issue at:

- [GreengageDB/greengage-ci](https://github.com/GreengageDB/greengage-ci)

For Greengage Database development, refer to:

- [GreengageDB/greengage](https://github.com/GreengageDB/greengage)
