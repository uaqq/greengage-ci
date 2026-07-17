# Publish packages to S3 repository action

This action publishes already-downloaded package files to an S3-backed repository via repomanager.

## Purpose

This action contains the shared "publish" logic used by any workflow that has already obtained package files by some means (e.g. downloaded release assets) and just needs them pushed into the S3-backed apt repository: check what's already published, import the GPG signing key, and run repomanager for the rest.

See `greengage-sync-packages.yml` for an example caller: it downloads `.deb`/`.ddeb` files attached to a GitHub release and hands the directory to this action.

## Prerequisites

Configure the following repository secrets:

### Secrets

Name                    | Description                                        | Required
----------------------- | -------------------------------------------------- | --------
`ghcr_login`            | GitHub login for GHCR access                       | Yes
`ghcr_password`         | GitHub Personal Access Token (PAT) for GHCR access | Yes
`aws_endpoint_url`      | S3 bucket endpoint URL                             | Yes
`aws_access_key_id`     | S3 bucket access key ID                            | Yes
`aws_secret_access_key` | S3 bucket secret access key                        | Yes
`gpg_private_key`       | GPG private key to sign repository                 | Yes

## Usage

```yaml
- name: Publish downloaded packages
  uses: GreengageDB/greengage-ci/.github/actions/publish-to-s3-repo@CI-5846
  with:
    packages_dir:           downloaded-packages
    s3_prefix:               'ubuntu/22.04/x86_64'
    aws_bucket:              ${{ secrets.BUCKET }}
    aws_endpoint_url:        ${{ secrets.AWS_ENDPOINT_URL }}
    aws_access_key_id:       ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws_secret_access_key:   ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    ghcr_login:              ${{ secrets.GHCR_LOGIN }}
    ghcr_password:           ${{ secrets.GHCR_PASSWORD }}
    gpg_private_key:         ${{ secrets.GPG_PRIVATE_KEY }}
```

With optional parameters:

```yaml
- name: Publish downloaded packages
  uses: GreengageDB/greengage-ci/.github/actions/publish-to-s3-repo@CI-5846
  with:
    packages_dir:            downloaded-packages
    extensions:               'deb ddeb'
    s3_prefix:                'ubuntu/22.04/x86_64'
    repomanager_image:        'ghcr.io/<repo>/<package>:<tag>'
    apt_codename:              'jammy'
    apt_component:             'contrib'
    aws_bucket:                ${{ secrets.BUCKET }}
    aws_endpoint_url:          ${{ secrets.AWS_ENDPOINT_URL }}
    aws_access_key_id:         ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws_secret_access_key:     ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    ghcr_login:                ${{ secrets.GHCR_LOGIN }}
    ghcr_password:             ${{ secrets.GHCR_PASSWORD }}
    gpg_private_key:           ${{ secrets.GPG_PRIVATE_KEY }}
```

### Inputs

Name                    | Description                                                                                    | Required | Type   | Default
----------------------- | ---------------------------------------------------------------------------------------------- | -------- | ------ | -------------------------------------------------------
`packages_dir`          | Local directory containing the package files to publish                                        | Yes      | String | -
`extensions`            | Space-separated list of package file extensions to look for inside `packages_dir`              | No       | String | `deb ddeb`
`s3_prefix`             | S3 bucket prefix for uploading packages (e.g. `ubuntu/22.04/x86_64`)                           | Yes      | String | -
`repomanager_image`     | Repomanager container image name (e.g 'ghcr.io/greengagedb/repomanager-ng-tool-x86_64:latest') | No       | String | `ghcr.io/greengagedb/repomanager-ng-tool-x86_64:latest`
`aws_bucket`            | S3 bucket name                                                                                 | Yes      | String | -
`aws_endpoint_url`      | S3 bucket endpoint URL (e.g. `https://storage.googleapis.com`)                                 | Yes      | String | -
`aws_access_key_id`     | S3 bucket access key ID                                                                        | Yes      | String | -
`aws_secret_access_key` | S3 bucket secret access key                                                                    | Yes      | String | -
`gpg_private_key`       | GPG private key to sign repository                                                             | Yes      | String | -
`apt_codename`          | Distribution codename used in the repository path (e.g. `bookworm`,`bullseye`,`jammy`)         | No       | String | greengagedb
`apt_component`         | Repository component/section name (e.g. `main`, `contrib`)                                     | No       | String | main
`ghcr_login`            | GitHub container registry login                                                                | Yes      | String | -
`ghcr_password`         | GitHub Personal Access Token (PAT)                                                             | Yes      | String | -

### Outputs

Name        | Description
----------- | -------------------------------------------------------
`published` | `"true"` if packages were uploaded, otherwise `"false"`

## What it does

1. Checks if the packages in `packages_dir` already exist in the repository.
2. Imports the GPG private key for repository signing.
3. Uploads the missing packages to the repository using the repomanager tool.
