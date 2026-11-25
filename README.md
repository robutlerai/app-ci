# app-ci

The reusable GitHub Actions workflow that releases a Robutler app from a version tag of its repository.

Push a tag such as `v3`. The workflow asks the Robutler portal to release the app from that tag. The portal reads
the tagged commit from GitHub itself, compares what the app is allowed to do with the current release, and releases
it. If the new version would let the app reach more (a new host, a browser permission, a new function), nothing is
released and the app's owner is asked to approve it in the portal.

## Use it

Your app needs a source repository linked in the portal (the app's source settings). Then add one file to the
repository, `.github/workflows/release.yml`:

```yaml
name: Release
on:
  push:
    tags: ['v*']

jobs:
  release:
    permissions:
      id-token: write
      contents: read
    uses: robutlerai/app-ci/.github/workflows/release.yml@v1
    with:
      app-id: 00000000-0000-4000-8000-000000000000
      environment: production
```

Pin `@v1`. A branch or a commit in place of the version tag is refused by the portal: the release job must be one
of the released versions of this workflow, so a repository cannot change what the job does.

## Inputs

| Input | Required | Meaning |
| --- | --- | --- |
| `app-id` | yes | The app's id in the portal |
| `environment` | yes | `dev`, `staging` or `production` |
| `credential` | no | `oidc` (default) or `token` |

| Secret | Needed for | Meaning |
| --- | --- | --- |
| `ROBUTLER_RELEASE_TOKEN` | `credential: token` | A release token from the portal: Settings, Developer, Release tokens |

| Output | Meaning |
| --- | --- |
| `outcome` | `released`, `unchanged` or `approval_requested` |
| `approval-url` | Where the owner approves, when the outcome is `approval_requested` |

## The two credentials

**`oidc`** needs no secret. GitHub signs a short identity token for the run and the portal checks that it is for
your app's linked repository, for a release tag, for the commit the tag is on, and for this workflow at a released
version. The calling job grants `id-token: write`. See `examples/caller-oidc.yml`.

**`token`** uses a release token you make in the portal and store as the repository secret
`ROBUTLER_RELEASE_TOKEN`. A release token can release apps and nothing else, names one app or every app you own,
lasts at most 90 days, and can be revoked in the portal at any time. See `examples/caller-token.yml`.

## Tags

A release tag is `v`, a digit, then lower case letters, digits, `.`, `_` or `-`: `v1`, `v1.2.0`, `v2-rc.1`.
A released tag does not move: cut a new tag for a new commit.

## What the job does not do

It checks out nothing and runs no code from your repository. It builds nothing: the tagged commit already holds
the files the app serves.

## Versions of this workflow

The portal accepts runs of this workflow at its released major version tags (`v1`). A new major version is
announced here before the portal accepts it.
