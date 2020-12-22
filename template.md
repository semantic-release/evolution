# Git Changelog Lib migration/fork proposal

Migrate [Git Changelog Lib](https://github.com/tomasbjerre/git-changelog-lib) to use same configuration, and features, as Semantic Release. To enable using same workflow with its [Gradle](https://github.com/tomasbjerre/git-changelog-gradle-plugin), [Maven](https://github.com/tomasbjerre/git-changelog-maven-plugin) and [Jenkins](https://github.com/jenkinsci/git-changelog-plugin) -plugins.

## Motivation

I am doing similar work as you do. But I do it for Java projects and Jenkins. I think everyone, users and developers, will gain from us joining forces.

## Proposed solution

I'm considering rewriting my [Git Changelog Lib](https://github.com/tomasbjerre/git-changelog-lib) to be compatible with your configuration.

I propose I work on a rewrite. When I'm ready, my repositories can be forked to the [semantic-release](https://github.com/semantic-release) organization.

## Detailed design

The library implementing core functionality: [Git Changelog Lib](https://github.com/tomasbjerre/git-changelog-lib)

Plugins that are kept as small as possible, just adding the glue needed to ease usage on those platforms:

- [Gradle plugin](https://github.com/tomasbjerre/git-changelog-gradle-plugin).
- [Maven plugin](https://github.com/tomasbjerre/git-changelog-maven-plugin).
- [Jenkins plugin](https://github.com/jenkinsci/git-changelog-plugin).
- [command line](https://github.com/tomasbjerre/git-changelog-command-line).

## Backward compatibility

No changes required on existing code in [semantic-release](https://github.com/semantic-release). I think I would like to keep existing features, and deprecate them, in the [Git Changelog Lib](https://github.com/tomasbjerre/git-changelog-lib).

## Alternatives considered

Alternative might be for me to do this work, but not expecting anything to be forked to [semantic-release](https://github.com/semantic-release).
