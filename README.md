# app-ci

The reusable GitHub Actions workflow that releases a Robutler app from a version tag of its repository.

Push a tag such as `v3`. The workflow asks the Robutler portal to release the app from that tag. The portal reads
the tagged commit from GitHub itself, compares what the app is allowed to do with the current release, and releases
it. If the new version would let the app reach more (a new host, a browser permission, a new function), nothing is
released and the app's owner is asked to approve it in the portal.

## Use it

Two things are set once in the portal, in the app's source settings, by the app's owner:

1. Link the app's source repository.
2. Turn on "Release from this repository's tags". It is off until the owner turns it on, and it is turned off
   again when the repository is linked again.

Then add one file to the repository, `.github/workflows/release.yml`:

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

The portal knows the linked repository by GitHub's own id for it, not by its name. If the repository is renamed,
moved to another account or replaced by a new one of the same name, link it again in the portal.

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

Releases move forward. A tag is released without a click only when it is a newer version than the tag the app's
current release came from (`v1.2.1` after `v1.2.0`, `v2` after `v1.9`). Running the workflow again for an older
tag, or for a tag the portal cannot place after the current release, releases nothing: the owner is asked to
approve it in the portal, exactly as for a version that would let the app reach more.

## What the job does not do

It checks out nothing and runs no code from your repository. It builds nothing: the tagged commit already holds
the files the app serves.

## Versions of this workflow

The portal accepts a run of this workflow only when both of these hold:

- the caller's `uses:` line names a released major version tag (`@v1`);
- the commit that tag is on is one of the commits the portal lists as reviewed.

The second check means that moving a version tag does not change what the portal accepts. A tag moved to a commit
the portal does not list is refused for every app until that commit is listed.

### How a new release of app-ci is admitted

1. The change to `release.yml` is reviewed and merged to `main` in this repository.
2. The full 40 character id of the merged commit is added to the portal's list of accepted commits
   (`APP_CI_WORKFLOW_SHAS` in `lib/casfs/github-oidc.ts`) in a reviewed change to the portal, next to the commits
   already there, and that portal change is deployed to every environment that accepts CI releases.
3. Only then is the version tag put on the commit: `v1` is moved for a compatible change, and a new major tag
   (`v2`) is created for a change callers must opt into. A new major tag is also added to the portal's list of
   accepted workflow versions (`APP_CI_WORKFLOW_REFS`) in step 2.
4. The commit the tag was on before stays in the portal's list until no run can still be using it, then it is
   removed.

Version tags in this repository are protected: only the maintainers of the Robutler organisation can create or
move them.
