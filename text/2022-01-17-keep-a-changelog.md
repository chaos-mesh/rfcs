# Keep A Changelog

## Summary

<!-- One para explanation of the proposal. -->

Proposal for keeping a changelog for https://github.com/chaos-mesh/chaos-mesh.

## Motivation

<!-- Why are we doing this? What use cases does it support? What is the expected
outcome? -->

It would consume a lot hand work to collect all the changes into release notes
before releasing a new version. It might be better to keep a changelog, commit
the changes for each PR.

Thanks to @yangkeao, he introduces us https://keepachangelog.com/ with best
practices.

## Detailed design

<!-- This is the bulk of the RFC. Explain the design in enough detail that:

- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.
- How the feature is used. -->

This proposal contains nothing about technical details and codes, it only
affects several steps about creating and reviewing PRs.

Most of following contents are copied from https://keepachangelog.com/, we would
follow the best practices and pattern from it.

### Guiding Principles

- Changelogs are for humans, not machines.
- There should be an entry for every single version.
- The same types of changes should be grouped.
- Versions and sections should be linkable.
- The latest version comes first.
- The release date of each version is displayed.
- Mention whether you follow [Semantic Versioning](https://semver.org/).

### Types of changes

- `Added` for new features.
- `Changed` for changes in existing functionality.
- `Deprecated` for soon-to-be removed features.
- `Removed` for now removed features.
- `Fixed` for any bug fixes.
- `Security` in case of vulnerabilities.

When creating a new `Unreleased` section, we would keep all the types of changes
with "Nothing", and trim empty entries when releasing a new version.

### When to create new items in the changelog

- As a contributor, when opening a new PR, if you think this PR should be
  considered in the changelog/release notes, please create a new item in the
  changelog.
- As a reviewer, if you think a PR without updating changelog is enough
  important for taking one, please ask the author of the PR to create one.
- As a contributor, if you find some a change is enough important for taking a
  changelog, but it doesn't, please open a new PR to update the changelog.
- As a reviewer/committer/maintainer, when you releasing a new version, sync the
  changelog with the released version.

### CHANGELOG.md in release-* branches

We should maintain a changelog for each active `release-*` branches, for now,
they are `release-2.0` and `release-2.1`. I would create a new file
`CHANGELOG.md` in each branch with the existing release notes. `CHANGELOG.md`
should also contains an `Unreleased` section for next patch/bugfix release.

When we cherry-pick a PR into `release-*` branch, if the original PR already
have items in `Unreleased` section, it could be also cherry-picked into here.

#### When releasing major, minor, or bugfix/patch version

The detailed step of creating a new release should be updated in release guide
`RELEASE.md`. We should manually change the `Unreleased` section to `[x.y.z] -
YYYY-MM-DD` in the `release-*` branch, and update `CHANGELOG.md` in `master`
branch manually.

So `CHANGELOG.md` in `release-*` only contains sections for minor and
bugfix/patch releases, and `CHANGELOG.md` in `master` contains sections for all
the releases.

As one principal is "The latest version comes first", and with semantic
versioning's comparing rule, `2.1.2 > 2.0.6`. So sections in `CHANGELOG.md` in
`master` should be like

```markdown

- [2.1.2] - 2021-12-29

(contents)

- [2.1.1] - 2021-12-10

(contents)

- [2.1.0] - 2021-11-30

(contents)

- [2.0.6] - 2021-12-29

(contents)

- [2.0.5] - 2021-11-25

(contents)

```

### What would be the first step

We would create a file called `CHANGELOG.md` in `master` branch, which content:

```markdown
# Chaos Mesh Changelog

(descriptions)

(guideline, link to this rfc)

## [Unreleased]

### Added

- some [#(pr-number)](https://link)
- entries [#(pr-number)](https://link) [#(pr-number)](https://link)
- (I would collect items from commits in master branch)

### Changed

- ditto

### Deprecated

(leave "Nothing" only in "Unreleased" section)
- Nothing

### Removed

- Nothing

### Fixed

- Nothing

### Security

- Nothing

## [2.1.2] - 2021-12-29

### Changed

- some [#(pr-number)](https://link)
- entries [#(pr-number)](https://link) [#(pr-number)](https://link)
- (I would collect items from existing release notes)

### Fixed

- ditto

## [2.0.6] - 2021-12-29

### Changed

- ditto

### Fixed

- ditto
```

And create `CHANGELOG.md` in each `release-*` branch with similar content.

## Drawbacks

<!-- Why should we not do this? -->

No predictable drawbacks could block this proposal now.

## Alternatives

<!-- - Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not
  choosing them?
- What is the impact of not doing this? -->

No other alternative solutions yet.

## Unresolved questions

<!-- What parts of the design are still to be determined? -->

Does Chaos Mesh follows Semantic Versioning?

I think Chaos Mesh does not follow Semantic Versioning now. We would release
major versions with more reasons about marketing, and we introduce API breaking
changes in minor releases. It does not matter now, because
https://github.com/chaos-mesh/chaos-mesh is not designed to be used as
dependency by other projects. I think we should mention that in the changelog.

About "Types of changes", which one would be better?

Pattern A:

- `Added` for new features.
- `Changed` for changes in existing functionality.
- `Deprecated` for soon-to-be removed features.
- `Removed` for now removed features.
- `Fixed` for any bug fixes.
- `Security` in case of vulnerabilities.

Pattern B:

- `New Features` for new features.
- `Enhancements` for changes in existing functionality.
- `Deprecated` for soon-to-be removed features.
- `Removed` for now removed features.
- `Fixed` for any bug fixes.
- `Security` in case of vulnerabilities.

I prefer Pattern A because a "change" might not be "enhancement". Maybe we could
mix them.
