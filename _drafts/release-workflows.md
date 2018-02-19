# Release workflows

Support multi-branch, multi-channel, LTS and pre-release workflows.

## Motivation

Support complex release workflows in a fully automated way, including:

- LTS branches and distribution channels
- Future branches and distribution channels
- Pre-releases

## Proposed solution

Support the following type of branches:
- [Single release branch](#single-release-branch)
- [Non release branch](#non-release-branch)
- [Future branches](#future-branches)
- [LTS branches](#lts-branches)
- [Pre-releases](#pre-releases)

## Detailed design

**Terminology:**

| Term                | Definition                                                                                                                                                                         |
|---------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Default branch      | Git branch that contains the stable code (made available to all users). Usually named `master`.                                                                                    |
| Future branches     | Permanent Git branch that contains the unstable code (made available only to early adopters). Usually named `next`.                                                                |
| Channel             | Distribution channels used to limit the accessibility of a release to a subset of users. For example `dist-tags` in `npm`.                                                         |
| Default channel     | Distribution channel to make stable version available to every users. Usually named `latest`.                                                                                      |
| Future channels     | Distribution channels to make unstable version available to early adopters. Usually named `next`.                                                                                  |
| Pre-release version | version that doesn't follow the regular `semver` increment pattern. For example any increment of `2.0.0-beta.1` is `2.0.0-beta.2`, then `2.0.0-beta.2` etc...                      |
| Version             | version following the `semver` rules. For example, a `minor` increment of `1.0.0` is `1.1.0`.                                                                                      |
| Module              | A software meant to be used as a dependency of other software. It's not meant to be used by final users.                                                                           |
| Application         | A software meant to be used by final users. No other software depends on it.                                                                                                       |                                                         |

### Branch types

#### Single release branch

Simple workflow publishing release for each commits on the `master` branch.

Ideal for small or stable modules receiving few breaking changes.

Branch: `master`

Channel: `latest` if supported by the release target

##### Single release branch workflows

| Action                               | Result                                                          |
|--------------------------------------|-----------------------------------------------------------------|
| **push** to `master`                 | Release on default channel (if supported, otherwise no channel) |
| **merge** feature branch to `master` | Release on default channel (if supported, otherwise no channel) |

##### Single release branch configuration

```json
{
  "release": {
    "branches": ["master"]
  }
}
```

#### Non release branch

Simple workflow publishing a release when merging `dev` into `master` branch. Pushing commits to `dev` does not trigger a release.

Ideal for modules in early stage development or during a development period with frequent breaking changes that doesn't need to be made available to users right away. This workflow allow to limit the release frequency by grouping multiple commits in one release.  

Branch: `master`, `dev`

Channel: `latest` if supported by the release target

##### Non release branch Workflow

| Action                               | Result                                                          |
|--------------------------------------|-----------------------------------------------------------------|
| **push** to `master`                 | Release on default channel (if supported, otherwise no channel) |
| **merge** feature branch to `master` | Release on default channel (if supported, otherwise no channel) |
| **push** to `dev`                    | -                                                               |
| **merge** feature branch to `dev`    | -                                                               |
| **merge** `dev` branch to `master`   | Release on default channel (if supported, otherwise no channel) |

##### Non release branch configuration

```json
{
  "release": {
    "branches": ["master"]
  }
}
```

#### Future branches

Channel based workflow to release on different channels. Make future release available on `latest` channel when merging associated branches into `master` branch.

Ideal for stable modules, distributing new features to a subset of users as soon as possible. This workflow allows to get feedback from early adopters while limiting the risk of regression for other users.

Branch: `master`, `next`

Channel: `latest`, `next`

##### Future branches workflow

| Action                                                | Result                                                                                                                                                                                                                                                                                                    |
|-------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **push** to `master`                                  | Release on `latest` channel if the commit trigger an allowed release type, report an error otherwise                                                                                                                                                                                                      |
| **merge** feature branch to `master`                  | Release on `latest` channel if the commit trigger an allowed release type, report an error otherwise                                                                                                                                                                                                      |
| **push** to `next`                                    | Release on `next` channel                                                                                                                                                                                                                                                                                 |
| **merge** any branch to `next`                        | Release on `next` channel                                                                                                                                                                                                                                                                                 |
| **merge** some commits from `next` branch to `master` | Get the last release for commits on branch `master` (including the one that come from `next`), analyze commits, increase the version; If the commit associated with the last release found is also present on `next`, make the version available on the `latest` channel, otherwise do a regular release. |

##### Future branches configuration

```json
{
  "release": {
    "branches": ["master", "next"]
  }
}
```

Internally **semantic-release** will determines the type of commit that can be done on each branch, based on the number of future branches defined:
- `"branches": ["master"]` => Any type of commit can be pushed to `master`
- `"branches": ["master", "next"]` => `master` will accept `fix` and `feat` commits; `next` will accept any type of commits.
- `"branches": ["master", "next", "next-major"]` => `master` will accept `fix`; `next` will accept `fix` and `feat` commits; `next-major` will accept any type of commits.

Those rules are meant to prevent inconsistent state in releases in which the same releases line is "forked".

For example with the config `"branches": ["master", "next"]` and:
- last release from `master` is `1.1.0`
- last release from `next` is `2.0.0`

Committing a breaking change to `master` would trigger the release of `2.0.0` that would fail as the release already exists.

In addition when the `master` and `next` have the same commit head (after a merging `next` into `master` for example), the first commit done on `next` must be a breaking change.
Otherwise that would result in a "fork" situation.

For example with the config `"branches": ["master", "next"]` and:
- last release from `master` is `2.0.0`
- last release from `next` is `2.0.0`

A `feat` commit on `master` will release version `2.1.0`.
Committing a `feat` commit on `next` would trigger the release of `2.1.0` that would fail as the release already exists.

With specific channel names:

```json
{
  "release": {
    "branches": [
      "master",
      {"branch": "next", "channel": "preview"}
    ]
  }
}
```

#### LTS branches

Workflow to release legacy versions.

Ideal for modules maintaining legacy versions and doing bug fixes and features only releases.

Branch: `1.x`, `2.x`, `master` (any number of lts branches are supported)

Channel: `1.x`, `2.x`, `latest` if supported by the release target

##### LTS branches workflow

| Action                                          | Result                                                                                                                  |
|-------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| **push** to `master`                            | Release on default channel (if supported, otherwise no channel)                                                         |
| **merge** feature branch to `master`            | Release on default channel (if supported, otherwise no channel)                                                         |
| **push** to `1.x` or `2.x` branches             | Release on `1.x` or `2.x` channel (if supported, otherwise no channel) if in range, report an error and error otherwise |
| **merge** any branch to `1.x` or `2.x` branches | Release on `1.x` or `2.x` channel (if supported, otherwise no channel) if in range, report an error and error otherwise |

##### LTS branches configuration

```json
{
  "release": {
    "branches": ["1.x", "2.x", "master"]
  }
}
```

With specific channel names:

```json
{
  "release": {
    "branches": [
      {"branch": "v1", "range": "1.x", "channel": "v1"},
      {"branch": "v2", "range": "2.x", "channel": "v2"},
      "master"
    ]
  }
}
```

#### Pre-releases

Workflow to distribute releases without incrementing the semantic version during the development of new a application version.  
Ideal for applications distributing unstable/alpha/preview releases.

Branch: `master`, `beta`, `alpha`

Channel: `latest` if supported by the release target

##### Pre-releases workflow

| Action                                              | Result                                                                                                                                                                                                                                                                                                                                                                                |
|-----------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **push** to `master`                                | Release on default channel (if supported, otherwise no channel)                                                                                                                                                                                                                                                                                                                       |
| **merge** `beta`, `alpha` or any branch to `master` | Release on default channel (if supported, otherwise no channel)                                                                                                                                                                                                                                                                                                                       |
| **push** to `beta`                                  | Get the last release for commits on branch `beta`; if the last release is also present on a lower branch (it would be the first release on `beta`), increment the version based on new commits and make a prelease on `beta` channel; if the last release is a pre-release increment the prerelease version (`4.0.0-beta.1`, then `4.0.0-beta.2`) and release on `beta` channel       |
| **merge** any branch to `beta`                      | Get the last release for commits on branch `beta`; if the last release is also present on a lower branch (it would be the first release on `beta`), increment the version based on new commits and make a prelease on `beta` channel; if the last release is a pre-release increment the prerelease version (`4.0.0-beta.1`, then `4.0.0-beta.2`) and release on `beta` channel       |
| **push** to `alpha`                                 | Get the last release for commits on branch `alpha`; if the last release is also present on a lower branch (it would be the first release on `alpha`), increment the version based on new commits and make a prelease on `alpha` channel; if the last release is a pre-release increment the prerelease version (`5.0.0-alpha.1`, then `5.0.0-alpha.2`) and release on `alpha` channel |
| **merge** any branch to `alpha`                     | Get the last release for commits on branch `alpha`; if the last release is also present on a lower branch (it would be the first release on `alpha`), increment the version based on new commits and make a prelease on `alpha` channel; if the last release is a pre-release increment the prerelease version (`5.0.0-alpha.1`, then `5.0.0-alpha.2`) and release on `alpha` channel |

##### Pre-releases configuration

```json
{
  "release": {
    "branches": [
      "master",
      {"branch": "beta", "prerelease": true},
      {"branch": "alpha", "prerelease": true},
    ]
  }
}
```

#### Configuration

##### Format

Each branch is either a `String` or an `Object` with the following properties:

| Option       | Description                                                                                                                                                             |
|--------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `branch`     | **Require** Git branch name                                                                                                                                             |
| `range`      | Accepted version range for to release from this branch. If defined the branch will be considered a LTS branch.                                                          |
| `channel`    | Channel on which to release. Ignored for release targets that don't support channels.                                                                                   |
| `prerelease` | Version of the pre-release done from this branch. Must be formatted `<version>-<tag>`. The version is required, the `tag` is optional. Examples: `4.0.0-beta`, `4.0.0`. |

**The order in which the branches are defines is meaningful, and it is used to determine the branch type and which release can be done from each branch.**

##### Branches naming convention

In a branch is defined as a `String` `semantic-release` will automatically determine the type of branch as follow:
- If the branch name is a valid semver range (Only the format `1.x.x` or `1.x` are valid Git reference names) the branch will be considered a LTS branch, with its range based on the branch name, and release on the channel with the same name
- The first branch in the `branches` `Array` after the LTS branches is considered the default branch (usually `master`).
- Each branches in the `branches` `Array` after the default branch are considered future branches.

For example:
```json
{
  "release": {
    "branches": [
      "1.x.x",
      "master",
      "next",
      {"branch": "alpha", "prerelease": true}
    ]
  }
}
```
is equivalent to:
```json
{
  "release": {
    "branches": [
      {"branch": "1.x.x", "range": "1.x.x", "channel": "latest"},
      {"branch": "master", "channel": "latest"},
      {"branch": "next", "channel": "next"},
      {"branch": "alpha", "channel": "alpha", "preid": "alpha", "prerelease": true}
    ]
  }
}
```

##### Configuration validation

The `branches` configuration must follow these rules:
- All branch name must be valid [Git reference](https://git-scm.com/docs/git-check-ref-format)
- A branch cannot be named after a valid semver version (for example `1.0.0` or `v1.0.0` are invalid branch names) or a name that match the `tagFormat`.
- A default branch is required.
- LTS branches range must not overlap among themselves and the default one. For example `["1.x.x", 1.5.x]` is invalid.
- LTS branches must be defined before the default branch.
- Futures branches must be defined after the default branch.
- A maximum of 2 future branch can be defined (one for features and one for breaking change)
- Default and future branches cannot define a `range`.

## Backward compatibility

Non backward compatible changes:
- Re-purpose the `branch` options as the current branch on which `semantic-release` is running
- Pass the `channel` on which to release `nextRelease` to the `publish` plugins
- publish plugins modifications
  - If release already exists, make it available on `nextRelease.channel` (in case of github, remove the `prerelease` flag if the target channel is the default one)
  - For the `npm` plugin, always publish on the dist-tag passed in `nextRelease.channel` or `latest` if `nextRelease.channel` is not set
  - For the `github` plugin, set `prerelease` flag if the current branch is a future or prerelease
