---
layout: post
title: Decouple Maven Releases from Version Logic for CI/CD
tags: [java, maven, ci/cd]
---

In standard Maven projects, versions are hardcoded in the POM, requiring a manual update after every release. This workflow is prone to several issues that often complicate CI/CD pipelines. This article explains why this approach is problematic and shows a better way to handle releases by decoupling the version and Git operations from Maven.

Let's first revise what Maven offers out of the box, namely the `maven-release-plugin` workflow:

1. **Transform**: Change version from 1.0-SNAPSHOT to 1.0.
2. **Commit & Tag**: Commit the change and create a Git release tag.
3. **Deploy**: Build and push the tagged version to the repository.
4. **Iterate**: Bump the version to 1.1-SNAPSHOT and commit again.

### Issues with Maintaining Hardcoded Versions in POM

#### Noise in Git History
Tags are required as they mark a point in Git history where we can reliably track releases. Version change commits, on the other hand, are not used in any context and add noise.

#### Merge Conflicts
Regardless of your branching strategy, you will eventually need to merge branches, like bringing a hotfix into the main branch. Because both branches have different hardcoded versions, an immediate merge conflict is triggered.

There are some options to work around this like cherry-picking and merge strategies among others, but that adds significant complexity to the CI automation.

#### Security Issues
Best practices dictate that main branches should be protected, preventing direct pushes, deletions, or commits without a pull request. In mature projects, CI/CD automation typically handles releases, creating a problem: you must either manually approve every version-change PR or whitelist a bot to bypass branch protection rules. Both are poor options: the first defeats the purpose of automation, while the second poses security risks like self-escalation via workflow edits.

### The Solution: Decouple Versioning and Git Operations from Maven
Analyzing the initial workflow, it is clear that Maven has two distinct roles: release orchestration and artifact deployment. To solve this, we need to separate the responsibilities between two actors:
1. **CI/CD**: Acting as the orchestrator of the release process.
2. **Maven**: Building and publishing the artifact.

While the following example focuses on a multi-module project, these steps are equally valid for standard projects. You can view the full implementation in this PR [https://github.com/apulbere/crop/pull/4/files](https://github.com/apulbere/crop/pull/4/files), but let's break down each step:

#### 1. Update the POMs to Use CI-Friendly Properties
Update the POMs to use `${revision}` instead of hardcoded versions. This should be done for the parent as well as children POMs.

Instead of:

```xml
<groupId>md.adrian</groupId>
<artifactId>crop-parent</artifactId>
<version>0.1.3</version>
```
we use:
```xml
<groupId>md.adrian</groupId>
<artifactId>crop-parent</artifactId>
<version>${revision}</version>
```

#### 2. Define a Default Version in the Parent POM
Defining a default version is necessary for local builds and IDEs to avoid errors. It can be overridden via the command line argument `-Drevision=0.1.5-SNAPSHOT`.
```xml
<properties>
    <revision>0.1.4-SNAPSHOT</revision>
</properties>
```

#### 3. Add the Flatten Maven Plugin
This is done for two reasons: to resolve `${revision}` and to remove everything that is not needed for the consumers of the artifact. The plugin actually creates other POMs. There are a couple of flatten modes, the most important ones:

* **resolveCiFriendliesOnly**: Useful for CI/CD pipelines, like running tests. It will only resolve `${revision}` in our case, but the rest will remain as is.
* **ossrh**: Useful for flattening when publishing to Maven Central. It removes everything that the consumer of the artifact does not actually use (parent references, build plugins, profiles, etc.) and creates a clean, minimal POM.

The published POM will look like this:

```xml
<groupId>md.adrian</groupId>
<artifactId>crop</artifactId>
<version>1.0.0</version>  <dependencies>
    <dependency>
        <groupId>jakarta.persistence</groupId>
        <artifactId>jakarta.persistence-api</artifactId>
        <version>3.1.0</version>
    </dependency>
</dependencies>
```

#### 4. Update .gitignore
Add `.flattened-pom.xml` to your `.gitignore`, so that you don't accidentally commit automatically generated flattened POMs.

#### 5. Update the CI/CD Pipeline
Amend the Maven deploy command to pass the revision version.
```bash
mvn deploy -Drevision=${{ steps.extract_version.outputs.version }}
```

In this example, the version is pulled directly from the GitHub Release action, which says both what version is used and when the release occurs. Once these responsibilities are separated, it is easy to extend the logic, for instance, by implementing automatic version calculation similar to to [jgitver](https://github.com/jgitver/jgitver). But mind you: in your CI/CD pipeline, not in Maven.