# gradle-android-git-version
[![Build Status](https://api.travis-ci.org/gladed/gradle-android-git-version.svg)](https://travis-ci.org/gladed/gradle-android-git-version) [![CircleCI](https://circleci.com/gh/gladed/gradle-android-git-version/tree/master.svg?style=svg)](https://circleci.com/gh/gladed/gradle-android-git-version/tree/master)

A gradle plugin to calculate Android-friendly version names and codes from git tags.

If you are
tired of manually updating your Android build files for each release, or generating builds that you
can't trace back to code, then this plugin is for you!

## Installation and Use

Add the plugin the top of your `app/build.gradle` (or equivalent):
```groovy
plugins {
    id 'com.gladed.androidgitversion' version '0.3.0'
}
```

Set `versionName` and `versionCode` from plugin results:
```groovy
android {
    // ...
    defaultConfig {
        // ...
        versionName androidGitVersion.name()
        versionCode androidGitVersion.code()
```

Use a git tag to specify your version number (see [Semantic Versioning](http://semver.org))
```bash
$ git tag 1.2.3
$ gradle --quiet androidGitVersion
androidGitVersion.name	1.2.3
androidGitVersion.code	1002003
```

For more advanced usage, read on...

## Tag format

Tags should look like `1` or `3.14` or `1.12.5`. There must be at least one tag of this format
reachable from the current commit, or the generated version name will be `"unknown"`.

Any suffix after the version number in the tag (such as `-release4` in `1.2.3-release4`) is included
in the version name, but is ignored when generating the version code.

In some cases, you'll have more than one version being generated in a single repo. In this case
you can supply a prefix like "lib-" to separate sets of tags. See `prefix` below for more.

## Intermediate Versions

Builds from non-tagged commits will generate a `name()` something like this:

`1.2.3-2-93411ff-fix_issue5-dirty`

In the example above, the components are:

| 1.2.3 | -2 | -93411ff | -fix_issue5 | -dirty |
| --- | --- | --- | --- | --- |
| Most recent tag | Number of commits since tag | Commit SHA prefix | Branch name | Dirty indicator |

Branches listed in `hideBranches` won't appear.

"-dirty" will only appear if the current branch has uncommitted changes.

You can customize this layout with `format` (see below).

## Version Codes

Version codes are calculated relative to the most recent tag. For example, version 1.2.3 will have
a version code of `1002003`.

You can customize the scheme used to generate version codes with `codeFormat` (see below).

## Methods

`name()` returns the current version name.

`code()` returns the current version code.

`flush()` flushes the internal cache of information about the git repo, in the event you have a
gradle task that makes changes to the repo.

# Tasks

`androidGitVersion` prints the name and code, as shown above.

`androidGitVersionName` prints only the name.

`androidGitVersionCode` prints only the code.

## Configuration Properties

An `androidGitVersion` block in your project's `build.gradle` file can supply optional properties
to configure this plugin's behavior, e.g.:

```groovy
androidGitVersion {
    baseCode 2000
    codeFormat = 'MNNPPP'
    format = '%tag%%.count%%<commit>%%-branch%%...dirty%'
    hideBranches = [ 'develop' ]
    onlyIn 'my-library'
    prefix 'lib-'
    untrackedIsDirty = false
}
```

### baseCode (int)
`baseCode` sets a floor for all generated version codes (that is, it is added to all generated
version codes). Use this when you have already released a version with a code, and don't want to go
backwards.

The default baseCode is 0.

### codeFormat (string)
`codeFormat` defines a scheme for building the version code. Each character corresponds to a
reserved decimal place in the resulting code:

- `M` for the Major version number (1.x.x)
- `N` for the Minor version number (x.1.x)
- `P` for the Patch version number (x.x.1)
- `B` place for the build number (revisions since last tag)

Take care; changing the version code scheme for a released Android project can cause problems if
your new version code does not
[increase monotonically](http://developer.android.com/tools/publishing/versioning.html). Consider
`baseCode` if you are changing code formats from a prior release.

Android version codes are limited to a maximum version code of 2100000000. As a result, codeFormat
only allows you to specify 9 digits.

The default is `codeFormat` is `"MMMNNNPPP"`, leaving 3 digits for each portion of the semantic
version.

### hideBranches (list of strings)
`hideBranches` describes which branches which should *not* be mentioned explicitly when building
intermediate versions (that is, versions without a tag). This will result in cleaner intermediate
version names when the branch name is obvious.

Note that each element of hideBranches is interpreted as a regex, for example, `[ 'master',
'feature/.*' ]`.

The default hideBranches are `[ 'master', 'release' ]`, meaning that intermediate builds will not
show these branch names.

### format (string)
`format` defines the form of the version name.

Parts include:
 - `tag` (the last tag)
 - `count` (number of commits, if any, since last tag)
 - `commit` (most recent commit prefix, if any, since the last tag)
 - `branch` (branch name, if current branch is not in `hideBranches`)
 - `dirty` (inserting the word "dirty" if the build was made with
uncommitted changes).

Parts are delimited as `%<PARTNAME>%`. Any other characters appearing between % marks are preserved.

Parts are sometimes omitted (such as a branch name listed by `hideBranches`). In this case the
entire part will not appear.

The default format is `"%tag%%-count%%-commit%%-branch%%-dirty%"`

### Deprecated: multiplier (int)
Use `codeFormat` instead.

`multiplier` sets the space allowed each part of the version when calculating the version code.

For example, if you want version 1.2.3 to have a version code of 100020003 (allowing for 9999 patch
increments), use `multiplier 10000`.

Use caution when increasing this value, as the maximum version code is 2100000000 (the maximum
integer).

The default multiplier is `1000`.

### onlyIn (string)
`onlyIn` sets a required path for relevant file changes. Commits that change files in this path
will count, while other commits will be ignored for versioning purposes.

For example, consider this directory tree:
```
+-- my-app/
    +-- .git/
    +-- build.gradle
    +-- app/
    |   +-- build.gradle
    |   +-- src/
    +-- lib/
        +-- build.gradle
        +-- src/
```
If `my-app/lib/build.gradle` is configured with `onlyIn 'lib'`, then changes to files in other
paths (like `my-app/build.gradle` or `my-app/app/src`) will not affect the version name.

The default onlyIn path is `''`, which includes all paths.

### Deprecated: parts (int)
Use `codeFormat` instead.

`parts` sets the number of parts the version number will have.

For example, if you know your product will only ever use two version number parts (1.2) then use
`parts 2`.

Use caution when increasing this value, as the maximum version code is 2100000000 (the maximum
integer).

The default number of parts is 3.

### prefix (string)
`prefix` sets the required prefix for any relevant version tag. For example, with `prefix 'lib'`,
the tag `lib-1.5` is used to determine the version, while tags like `1.0` and `app-2.4.2` are
ignored.

The default prefix is `''`, which matches all numeric version tags.

### untrackedIsDirty (boolean)
When `untrackedIsDirty` is true, a version is considered dirty when any untracked files are
detected in the repo's directory.

The default setting is `false`; only tracked files are considered when deciding on "dirty".

## License

All code here is Copyright 2015-2016 by Glade Diviney, and licensed under the
[Apache 2.0 License](http://www.apache.org/licenses/LICENSE-2.0).
