# [Draft] Artifact deprecation

Distribution of this memo is unlimited.

## Overview

We propose a formal mechanism that allows maintainers to deprecate a Maven artifact in a repository. The deprecation indicator will let downstream consumers know that the artifact is no longer actively developed and should not be used.

The Maven CLI tools could then react to deprecation indicators in the appropriate ways:

- `mvn` itself: Print a warning when deprecated dependencies are seen.
- Maven Enforcer Plugin: Add a `<banDeprecatedDependencies>` rule which throws an error when deprecated dependencies are seen.
- Maven Dependency Tree: Print a `[deprecated]` notice next to any deprecated dependency in the tree.
- ...and so on

We can also envisage automated agents like GitHub's Dependabot which consume these indicators, alert developers about deprecated dependencies in their stacks, and even help developers to remove them.

With these tools acting together, the community will be able to remove deprecated dependencies from significant numbers of projects. This will improve the health of the Maven ecosystem at large.

## Rationale

The Maven ecosystem contains some artifacts which were abandoned by their maintainers years ago, but which live on well past their sell-by dates.

The continued use of these artifacts causes problems like:

- Security risks
  - Vulnerabilities are found in old artifacts, but with no active maintainer they will never be fixed.
  - Malicious maintainers can take over old widely-used artifacts to insert backdoors.
- Holding back upgrades
  - The presence of old artifacts (and their transitive dependencies) in a dependency graph can stop a team upgrading their downstream application to newer library or language versions.

At the moment, the best a conscientious maintainer can do is to put a note about deprecation in their README or write a blog post, and hope that at least some users stumble across it. Let's give maintainers a better way to deprecate.

## Definitions

### What is deprecation?

Our definition of 'deprecated' is roughly 'this artifact is not actively developed and it should not be used any more'.

Its counterpart is undeprecation - the removal of the deprecation notice at a later date.

### What is not deprecation?

We consider deprecation to be different from the following concepts, and therefore do not intend to include them within the deprecation feature.

#### 'Alternatives'

Some of the other build tools (Nuget, Packagist, Cocoapods) include alternative package recommendations in their deprecation indicators.

However deprecation is largely orthogonal to alternatives:

- Alternatives exist at all stages of an artifact's lifecycle, not just during deprecation.
- Some artifacts are one-of-a-kind and have no alternatives.
- Developers may need to use an alternatives system at any stage of an artifact's lifecycle, not just during deprecation. (E.g. a company discovers that an artifact has a disallowed license and asks its developers to find an alternative.)
- Alternatives themselves come and go, so if an artifact maintainer recommends an alternative in a deprecation notice, this recommendation may become outdated. There may also be a better alternative that the maintainer does not know about.

Therefore an alternatives feature would be better as a sibling of the deprecation feature, not embedded within it.

#### Unpublishing or deletion

Deprecation should leave the artifact in the repository with a marker. It should not mean deleting or unpublishing the artifact altogether; those actions would break links.

#### Unlisting

Deprecation should not change whether the artifact appears in repository search results.

#### Changing visibility

Deprecation should not change who can access the artifact. If it was public before deprecation, it remains public.

#### Relinquishing ownership

In some repository systems, a maintainer can relinquish ownership of a package:

- In NPM they must [transfer it to the @npm account](https://docs.npmjs.com/deprecating-and-undeprecating-packages-or-package-versions#transferring-a-deprecated-package-to-npm), where it becomes permanently read-only and non-transferrable.
- In other systems the package may simply exist with nobody having write access, allowing the repo administrator to assign write access to new teams or maintainers in future.

Deprecation by itself should not mean giving up ownership. However that might be a next step after deprecation.

#### 'Read-only' or 'archived'

A maintainer should be able to provide new maintenance versions of an artifact under an overall deprecation banner. This allows maintainers to provide an extended support period under the deprecation banner, for users that can't migrate off the artifact immediately. It also means that new users won't mistake those maintenance updates for an actually-active artifact.

Example: Joda Time's maintainers release occasional timezone database updates to ensure it keeps working.

This is why we think deprecation should not make the artifact read-only; that would impose a single hard cut-off date for maintenance.

## Example

Joda Time is a date/time library which was superseded years ago by java.time in Java 8 and Android API 26. It has not been under active development for years, and now receives only timezone database updates.

Nevertheless, it's not going away - in fact it's spreading into more projects over time. GitHub says it had roughly 110,000 public dependents in Jan 2021, and 118,000 in July 2021. (The true figure is higher as this doesn't count private projects, or projects outside of GitHub.)

It's a textbook de-facto deprecated dependency, but at the moment, there is no formal mechanism to warn Joda's downstream consumers about this. The only indicator today is a deprecation note in the README.

## Where to apply a deprecation indicator?

There are 3 levels at which you could apply a deprecation indicator:

- Whole artifact
- Version range(s) - but Maven does not have these
- Individual version(s)

In practice the only approach that makes sense is to indicate deprecation at the **artifact level**. This is for a couple of reasons.

### Deprecation is a statement about future development activity, not about past versions

Deprecation says 'this artifact is not actively maintained, please stop using it and don't expect to see further changes in it'. (The maintainer might push out maintenance fixes occasionally, but users cannot count on this.)

This makes sense at the artifact level, because new versions can appear under an artifact. It does not make sense when applied to individual versions because released versions are immutable; they cannot be changed by definition.

### It enables connection to upstream source deprecations

A maintainer may want to connect the deprecation of an artifact to the deprecation of its upstream source code.

Upstream code is often deprecated by:

- Putting a note in the README
- Changing the repo description
- (GitHub) Setting the repo to 'Archived'

All of these upstream deprecation indicators effectively happen at the repo level, not at the level of individual versions.

In this context, artifact-level deprecation makes sense: 1 deprecated/archived Git repo = 1 deprecated Maven artifact. Meanwhile version-level deprecation does not make sense; Git commits or tags don't have deprecation markers.

### A per-version deprecation flag is redundant for encouraging version upgrades

Dependabot and IDEs already chase developers to update their dependency versions. (IntelliJ IDEA gained this ability in v2021.2.) If a newer version exists, the implication is that you should upgrade to it anyway. It would therefore be redundant to add another marker saying 'please upgrade from version X.Y.Z'.

## Setting a deprecation indicator

Now we consider the mechanism that allows people to set (and remove) a deprecation indicator.

### Who can deprecate?

At a minimum, the **maintainers** of the artifact must be able to deprecate it.

In practice, this changes a bit once you consider how the permission systems of Maven repositories like Nexus or Artifactory work. In those systems, anybody with permission to *manage* an artifact (this is more powerful than the permission to *deploy* artifact versions) will by extension be able to deprecate it. This means that **admins** can deprecate artifacts too.

To allow more fine-grained control, repositories could have *deprecate* and *undeprecate* permissions which are distinct from the *manage* permission.

### How to deprecate?

- Click a button in the repository Web UI.
- HTTP request to the repository API.
  - CLI wrapper: `mvn deprecate`?


## Requirements

General:

- A maintainer must be able to add a deprecation indicator to a published artifact.
- The deprecation feature must respect the immutability of the contents of published artifacts. (E.g. an artifact's POM cannot be modified after publication to add a deprecation indicator.)
- A maintainer should not have to publish a new version of an artifact just to (un)deprecate it. (In some cases it may not be possible to compile the artifact any more.)
- The deprecation indicator must be usable by:
  - Maven CLI
  - Maven-compatible build tools (E.g. Gradle, Leinginen, SBT.)
  - Third-party tools. (E.g. Dependabot, IntelliJ dependency inspection.)
- A maintainer must be able to undeprecate a deprecated artifact at a later date. (E.g. if deprecation was done by mistake.)

MVN:

- `mvn` should print a warning when deprecated dependencies are seen.

Maven Enforcer Plugin:

- The plugin should have a `<banDeprecatedDependencies>` rule which throws an error when deprecated dependencies are seen.
- The rule should have a `skip`-like property, to allow developers to toggle the error behavior off when needed.

Maven Dependency Tree Plugin:

- The plugin should print a `[deprecated]` notice next to any deprecated dependency in the tree.

## Comparison with other build tools

The following build tools currently have deprecation mechanisms:

- NPM: <https://docs.npmjs.com/cli/v7/commands/npm-deprecate>
- Nuget: <https://docs.microsoft.com/en-us/nuget/nuget-org/deprecate-packages>
- Composer: <https://tomasvotruba.com/blog/2017/07/03/how-to-deprecate-php-package-without-leaving-anyone-behind/>
- Cocoapods

Detailed descriptions of each one are below.

### NPM

The NPM `deprecated` property exists in NPM registry package metadata.

It is described here: <https://github.com/npm/registry/blob/master/docs/responses/package-metadata.md>

In NPM, deprecation is **per-version**. (NPM metadata lists all versions of a package. Each version object has an optional `deprecated` string property. This string is the deprecation warning message for that version. If the property is absent, NPM infers the package is still active.)

Example snippet:

    {
      "name": "<package-name>",
      "modified": "2017-03-21T21:40:18.939Z",
      "dist-tags": {
        "latest": "<semver-compliant version string>",
        "<dist-tag-name>": "<semver-compliant version string>"
      },
      "versions": {
        "<version>": {
            "name": ...,
            "deprecated": "please don't use this"    // this version is deprecated
        },
        "<version>": {
           "name": ...                               // this version is active
        }
      }
    }

The `npm deprecate` command essentially retrieves the package metadata with packument, sets the `deprecated` message on the specified version(s), and uploads it with packument.

The `npm outdated` command prints the message as a warning if it is seen.

### Nuget

The NuGet `deprecation` object exists in the NuGet registry package metadata.

It is described here: <https://docs.microsoft.com/en-us/nuget/api/registration-base-url-resource#package-deprecation>

In Nuget, deprecation is **per-version**. (Though a shortcut is provided to deprecate all versions, presumably because this is commonplace.)

Example snippet:

    {
      "@id": "https://api.nuget.org/v3/registration3/nuget.protocol.v3.example/1.0.729-unstable.json",
      "@type": "Package",
      "commitId": "e0b9ca79-75b5-414f-9e3e-de9534b5cfd1",
      "commitTimeStamp": "2017-10-26T14:12:19.3439088Z",
      "catalogEntry": {
        "@id": "https://api.nuget.org/v3/catalog0/data/2015.02.01.18.22.05/nuget.protocol.v3.example.1.0.729-unstable.json",
        "@type": "PackageDetails",
        "authors": "NuGet.org Team",
        "deprecation": {
          "reasons": [
            "CriticalBugs"
          ],
          "message": "This package is unstable and broken!",
          "alternatePackage": {
            "id": "Newtonsoft.JSON",
            "range": "12.0.2"
          }
        },
        "id": "NuGet.Protocol.V3.Example",
        "licenseUrl": "http://www.opensource.org/licenses/ms-pl",
        "packageContent": "https://api.nuget.org/v3-flatcontainer/nuget.protocol.v3.example/1.0.729-unstable/nuget.protocol.v3.example.1.0.729-unstable.nupkg",
        "projectUrl": "https://github.com/NuGet/NuGetGallery",
        "summary": "This package is an example for the V3 protocol.",
        "title": "NuGet V3 Protocol Example",
        "version": "1.0.729-Unstable"
      },
      "packageContent": "https://api.nuget.org/v3-flatcontainer/nuget.protocol.v3.example/1.0.729-unstable/nuget.protocol.v3.example.1.0.729-unstable.nupkg",
      "registration": "https://api.nuget.org/v3/registration3/nuget.protocol.v3.example/index.json"
      ...
    }

The `dotnet list package --deprecated` command shows deprecated packages in an application's dependency graph.

### Composer

The Composer `abandoned` property exists in Packagist's package metadata.

It can be a simple boolean flag:

    {
      "name": "phpunit/php-token-stream",
      "abandoned": true
    }

Or the maintainer can suggest an alternative package on Packagist:

    {
      "name": "phpunit/phpunit-story",
      "abandoned": "behat/behat",
    }

In Composer, deprecation is **per-artifact**. (The `abandoned` marker applies to all previously published versions of the artifact.)

Example snippet:

    GET https://repo.packagist.org/p2/phpunit/php-token-stream.json

    {
      "minified": "composer/2.0",
      "packages": {
        "phpunit/php-token-stream": [
          {
            "name": "phpunit/php-token-stream",
            "description": "Wrapper around PHP's tokenizer extension.",
            "version": "4.0.4",
            "version_normalized": "4.0.4.0",
            "require": {
              "php": "^7.3 || ^8.0",
              "ext-tokenizer": "*"
            },
            "require-dev": {
              "phpunit/phpunit": "^9.0"
            },
            "abandoned": true,
          },
          {
            "version": "4.0.3",
            "version_normalized": "4.0.3.0",
            "source": {
              "type": "git",
              "url": "https://github.com/sebastianbergmann/php-token-stream.git",
              "reference": "5672711b6b07b14d5ab694e700c62eeb82fcf374"
            },
            "dist": {
              "type": "zip",
              "url": "https://api.github.com/repos/sebastianbergmann/php-token-stream/zipball/5672711b6b07b14d5ab694e700c62eeb82fcf374",
              "reference": "5672711b6b07b14d5ab694e700c62eeb82fcf374",
              "shasum": ""
            },
          },
          {
            "version": "4.0.2",
            "version_normalized": "4.0.2.0",
            "source": {
              "type": "git",
              "url": "https://github.com/sebastianbergmann/php-token-stream.git",
              "reference": "e61c593e9734b47ef462340c24fca8d6a57da14e"
            },
            "dist": {
              "type": "zip",
              "url": "https://api.github.com/repos/sebastianbergmann/php-token-stream/zipball/e61c593e9734b47ef462340c24fca8d6a57da14e",
              "reference": "e61c593e9734b47ef462340c24fca8d6a57da14e",
              "shasum": ""
            },
            "require": {
              "php": "^7.3",
              "ext-tokenizer": "*"
            }
          },
          ...
        ]
      }
    }

### Cocoapods

Cocoapods lets the user put a deprecation property in the Podspec (the equivalent of the POM).

'Deprecated' (<https://guides.cocoapods.org/syntax/podspec.html#deprecated>):

    spec.deprecated = true

'Deprecated in favor of' (<https://guides.cocoapods.org/syntax/podspec.html#deprecated_in_favor_of>):

    spec.deprecated_in_favor_of = "<pod-name>"

Because a published artifact's contents cannot be modified, you must compile and publish a new version of a pod with this property to deprecate it.

TODO: find out whether it's possible to undeprecate a pod.

In Cocoapods, deprecation is **per-artifact**. (The `deprecated` marker applies to all previously published versions of the pod.)

## Comparison with similar Maven features

These existing Maven features have some overlap with the proposed feature, but are not the same.

### Relocation

<https://maven.apache.org/guides/mini/guide-relocation.html>

Hervé Boutemy says: relocation is far from well known and easy to use, but it is probably quite related.

### Old GroupIds Alerter

[Old GroupIds Alerter Maven Plugin](https://github.com/jonathanlermitage/oga-maven-plugin)

The plugin tracks the renaming of groupIds. It keeps a list of replaced/deprecated artifacts here: <https://raw.githubusercontent.com/jonathanlermitage/oga-maven-plugin/master/uc/og-definitions.json>.

This addresses a limited subset of deprecation where:

- an old artifact is replaced one-for-one by a new artifact
- the new artifact is the direct continuation of the old artifact

As such it is not a general deprecation feature.
