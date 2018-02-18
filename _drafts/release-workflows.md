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
"release": {
  "branches": ["master"]
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
"release": {
  "branches": ["master"]
}
```

#### Future branches

Channel based workflow to release on different channels. Make future release available on `latest` channel when merging associated branches into `master` branch.

Ideal for stable modules, distributing new features to a subset of users as soon as possible. This workflow allow to to get feedback from early adopters while limiting the risk of regression for other users.

Branch: `master`, `next`

Channel: `latest`, `next`

##### Future branches workflow

| Action                                                | Result                                                                                                                                                                                                                                                                                                    |
|-------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **push** to `master`                                  | Release on `latest` channel if the commit trigger a release with a version that doesn't satisfies the `type` defines for `next` branch, report an error otherwise                                                                                                                                         |
| **merge** feature branch to `master`                  | Release on `latest` channel if the commit trigger a release with a version that doesn't satisfies the `type` defines for `next` branch, report an error otherwise                                                                                                                                         |
| **push** to `next`                                    | Release on `next` channel                                                                                                                                                                                                                                                                                 |
| **merge** any branch to `next`                        | Release on `next` channel                                                                                                                                                                                                                                                                                 |
| **merge** some commits from `next` branch to `master` | Get the last release for commits on branch `master` (including the one that come from `next`), analyze commits, increase the version; If the commit associated with the last release found is also present on `next`, make the version available on the `latest` channel, otherwise do a regular release. |

##### Future branches configuration

```json
"release": {
  "branches": [
    "master", {
      "branch": "next",
      "type": "major"
    }
  ]
}
```

The `type` is required for future branches and indicate that its reserved for a certain type of release (`minor` or `major`).
That defines how much ahead the future branch must be versus the pervious branch in the list. If configured with `major`, the `next` branch must always one `major` release ahead of `master`.

Example 1:
- if the last release on `latest` is `2.1.0` and `next` is configured with `"type" : "major"`
- and the last release on `next` is also `2.1.0`
- then any releases are allowed on `latest`
- And only `major` releases are allowed on `next`

Example 2:
- if the last release on `latest` is `2.1.0` and `next` is configured with `"type" : "major"`
- and the last release on `next` is `2.5.0`
- then `patch` releases are allowed on `latest`, but `minor` and `major` are forbidden
- And any releases are allowed on `next`

Example 3:
- if the last release on `latest` is `2.1.0` and `next` is configured with `"type" : "major"`
- and the last release on `next` is `3.0.0`
- then `patch` and `minor` releases are allowed on `latest`, but `major` are forbidden
- And any releases are allowed on `next`

Example 4:
- if the last release on `latest` is `3.0.0` and `next` is configured with `"type" : "major"`
- and the last release on `next` is `2.5.0`
- And any releases are allowed on `latest`
- No releases are allowed on `next` (it's considered stalled) until `master` is merged into `next`. Once it's done the situation becomes the one described in Example 1

That enforces consistence across versions: **if a `x.y.z` version exists on a given channel, all superior versions on that channel must includes all the commits of `x.y.z`**

If not specified the `type` is `major`.

With specific channel names:

```json
"release": {
  "branches": [
    "master",
    {
      "branch": "next",
      "channel": "experimental",
      "type": "major"
    }
  ]
}
```

#### LTS branches

Workflow to release legacy versions.

Ideal for modules maintaining legacy versions and doing bug fixes and features only releases.

Branch: `1.x.x`, `2.x.x`, `master` (any number of lts branches are supported)

Channel: `latest` if supported by the release target

##### LTS branches workflow

| Action                                              | Result                                                                                                           |
|-----------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| **push** to `master`                                | Release on default channel (if supported, otherwise no channel)                                                  |
| **merge** feature branch to `master`                | Release on default channel (if supported, otherwise no channel)                                                  |
| **push** to `next`                                  | -                                                                                                                |
| **push** to `1.x.x` or `2.x.x` branches             | Release on default channel (if supported, otherwise no channel) if in range, report an error and error otherwise |
| **merge** any branch to `1.x.x` or `2.x.x` branches | Release on default channel (if supported, otherwise no channel) if in range, report an error and error otherwise |

Note: By default lts releases are published on the default channel as they are limited to a range of versions lower than the versions released from `master`. Dependents will get only the expected releases by specifying a range dependency (for example `^2.0.0`).
If a `channel` is specified, then the release will be made on this channel.

##### LTS branches configuration

```json
"release": {
  "branches": ["1.x.x", "2.x.x", "master"]
  }
}
```

With specific channel names:

```json
"release": {
  "branches": [{
    "branch": "v1",
    "range": "1.x.x",
    "channel": "v1"
  }, {
    "branch": "v2",
    "range": "2.x.x",
    "channel": "v2"
  }, "master"]
}
```

#### Pre-releases

Workflow to distribute releases without incrementing the semantic version during the development of new a application version.  
Ideal for applications distributing unstable/alpha/preview releases.

Branch: `master`, `next-beta`, `next-alpha`

Channel: `latest` if supported by the release target

##### Pre-releases workflow

| Action                                                         | Result                                                                                                                                        |
|----------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------|
| **push** to `master`                                           | Release on default channel (if supported, otherwise no channel)                                                                               |
| **merge** `4.0.0-beta`, `5.0.0-beta` or any branch to `master` | Release on default channel (if supported, otherwise no channel)                                                                               |
| **push** to `4.0.0-beta`                                       | Release a prerelease version (`4.0.0-beta`, then `4.0.0-beta.1`, then `4.0.0-beta.2`) on default channel (if supported, otherwise no channel) |
| **merge** any branch to `4.0.0-beta`                           | Release a prerelease version (`4.0.0-beta`, then `4.0.0-beta.1`, then `4.0.0-beta.2`) on default channel (if supported, otherwise no channel) |
| **push** to `5.0.0-beta`                                       | Release a prerelease version (`5.0.0-beta`, then `5.0.0-beta.1`, then `5.0.0-beta.2`) on default channel (if supported, otherwise no channel) |
| **merge** any branch to `5.0.0-beta`                           | Release a prerelease version (`5.0.0-beta`, then `5.0.0-beta.1`, then `5.0.0-beta.2`) on default channel (if supported, otherwise no channel) |

##### Pre-releases configuration

```json
"release": {
  "branches": ["master", "4.0.0-beta", "5.0.0-alpha"]
}
```

With specific branch names:

```json
"release": {
  "branches": ["master", {
    "branch": "dev",
    "prerelease": "4.0.0-alpha"
  }, {
    "branch": "experimental",
    "prerelease": "5.0.0-beta"
  }]
}
```

#### Configuration

##### Format

Each branch is either a `String` or an `Object` with the following properties:

| Option       | Description                                                                                                                                                             |
|--------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `branch`     | **Require** Git branch name                                                                                                                                             |
| `range`      | Accepted version range for to release from this branch. If defined the branch will be considered a LTS branch.                                                          |
| `channel`    | Channel on which to release. Ignored for release targets that don't support channels.                                                                                 |
| `prerelease` | Version of the pre-release done from this branch. Must be formatted `<version>-<tag>`. The version is required, the `tag` is optional. Examples: `4.0.0-beta`, `4.0.0`. |

**The order in which the branches are defines is meaningful, and it is used to determine the branch type and which release can be done from each branch.**

##### Branches naming convention

In a branch is defined as a `String` `semantic-release` will automatically determine the type of branch as follow:
- If the branch name is a valid semver range (`1.x.x`, `1.0.x`, `^1.0.0`) the branch will be considered a LTS branch, with its range based on the branch name, and release on the channel with the same name formatted `1.x.x` (`^1.0.0` => `1.x.x`)
- The first branch in the `branches` `Array` after the LTS branches is considered the default branch (usually `master`).
- Each branches in the `branches` `Array` after the default branch are considered future branches.
- If a future branch is named `minor`, `major`, `breaking` or `feature` it's type will be determined accordingly
- If the branch name is formatted like `<version>-<tag>` the branch will be considered a pre-release branch with `version` and `tag` based on the branch name.

For example:
```json
"release": {
  "branches": ["1.x.x", "master", "next", "5.0.0-alpha"]
}
```
is equivalent of:
```json
"release": {
  "branches": [{
    "branch": "1.x.x",
    "range": "1.x.x",
    "channel": "latest"
  }, {
    "branch": "master",
    "channel": "latest"
  }, {
    "branch": "next",
    "channel": "next"
  }, {
    "branch": "5.0.0-alpha",
    "prerelease": "5.0.0-beta"
  }]
}
```

##### Configuration validation

The `branches` configuration must follow these rules:
- A branch cannot be named after a valid, non pre-release semver version (for example `1.0.0` or `v1.0.0` are invalid branch names).
- A default branch is required.
- LTS branches range must not overlap. For example `["1.x.x", 1.5.x]` is invalid.
- LTS branches must be defined before the default branch.
- Futures branches must be defined after the default branch.
- Default and future branches cannot define a `range`.

## Deprecate `getLastRelease` and move tag management to the core

In order to determine the last version from which to calculate the new one, the `getLastRelease` need to identify the last version present on the current branch.

For example:
- The code corresponding to version `1.0.0` is present on `master`
- The code corresponding to version `1.1.0`, `1.2.0` and `1.3.0` is present on the `next` branch
- The user prepare a PR to `master` with the commits from `next` corresponding to version up to `1.2.0` and add a `fix` commit
- The PR get merged
- semantic-release runs on master and must:
  - retrieve the last version present on `master` which is `1.2.0` (even though `1.2.0` was published only from `next`)
  - then evaluate the commits between `1.2.0` and now
  - determine there is a new `fix` commit
  - increase the version to `1.2.1` and publish

Similarly if the user PR contains the commits from `next` corresponding to version up to `1.2.0` without additional commits, semantic-release will:
- retrieve the last version present on `master` which is `1.2.0` (even though `1.2.0` was published only from `next`)
- then evaluate the commits between `1.2.0` and now
- determine there is no new commit
- still call the publish plugins in order to make `1.2.0` available on the channel associated with `master`

This behavior will work by default with the `getLastRelease` of the git plugin, because when merging commits from `next` to `master` the tags will follow.
So when semantic-release runs on master the git plugin will by default retrieve the last version present on the branch as it uses tags.

This would be quite complex to implement for the npm plugin's `getLastRelease` though. We would need to:
- retrieve the last release on the dist-tag corresponding to `master`
- retrieve all the version released after that
- for each version get the githead
- check if the gitHead is on master
- return the version and gitHead corresponding to the highest version present on `master`

This is actually even more complex as GitHub allow to "Rebase and Merge" which rewrite the commit sha.
That means we would have for each gitHead, find the corresponding commit tree sha.

The only reliable and efficient way to obtain the last release would be to use git tags, which imply that each semantic-release creates a tag.

**Proposal**
- Deprecate the `getLastRelease` plugin
- Move the git plugin `getLastRelease` logic in to the Core
- Create the git tags from the Core as it's now mandatory to have a tag for each release

That would make semantic-release only works with Git repo. But it is already the case anyway.

It has the advantages of:
- allowing semantic-release to not rely on plugins behavior for tag creation
- Simplify the `publish` plugins has they would not have to deal with tag creation and handling the case where the tag already exists
- Centralize the complexity of last release retrieval to the core, making it easier to create simple, focused plugins
- As the authentication to the git repo will be handled and verified at the core level (via `GH_TOKEN` or `GIT_CREDENTIALS`), plugin can easily access the repo without having any extra complexity
- Relying on tag to identify version make it easier to "fix problem" as tag can be created easily, while gitHead on npm can't be changed, and is somewhat unreliable due to npm/read-package-json#77 and the fact it's a non documented feature

The inconvenient are:
- tags are now mandatory, but I can't think of a situation in which we wouldn't want to create a tag.
- IT wouldn't be possible to have a npm only config, that release on npm but doesn't create a tag in the repo, but I don't think it's something desirable anyway

## Backward compatibility

Non backward compatible changes:
- Re-purpose the `branch` options as the current branch on which `semantic-release` is running
- Pass the `channel` on which to release `nextRelease` to the `publish` plugins
- publish plugins modifications
  - If release already exists, make it available on `nextRelease.channel` (in case of github, remove the `prerelease` flag if the target channel is the default one)
  - For the `npm` plugin, always publish on the dist-tag passed in `nextRelease.channel` or `latest` if `nextRelease.channel` is not set
  - For the `github` plugin, set `prerelease` flag if the current branch is a future or prerelease
- `getLastRelease` deprecation
