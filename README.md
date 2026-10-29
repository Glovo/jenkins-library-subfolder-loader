<div align="center">

# 📚 jenkins-library-subfolder-loader

**A Jenkins step that loads a shared library from a subfolder of the repo being built.**

This repo is a one-step Jenkins **shared library**: it provides
[`loadLocalLibrary`](vars/loadLocalLibrary.groovy), which lets a Jenkinsfile use pipeline code kept in a
folder of its own repo. It is a fork of
[marcus-held/jenkins-library-subfolder-loader](https://github.com/marcus-held/jenkins-library-subfolder-loader)
with one Glovo change. It is not a pipeline, not a Jenkins plugin and not a published artifact: Jenkins
reads [`vars/`](vars) straight from git.

[![Groovy](https://img.shields.io/badge/Groovy-Jenkins_pipeline-4298B8?logo=apachegroovy&logoColor=white)](vars/loadLocalLibrary.groovy)
[![Jenkins core](https://img.shields.io/badge/Jenkins_core-2.201-D24939?logo=jenkins&logoColor=white)](build.gradle)
[![Gradle](https://img.shields.io/badge/Gradle-5.2.1_wrapper-02303A?logo=gradle&logoColor=white)](gradle/wrapper/gradle-wrapper.properties)
[![Java](https://img.shields.io/badge/Java-1.8_source-orange?logo=openjdk&logoColor=white)](build.gradle)
<br/>
[![Licence](https://img.shields.io/badge/licence-MIT-blue)](LICENCE)
[![Fork](https://img.shields.io/badge/fork_of-marcus--held%2Fjenkins--library--subfolder--loader-lightgrey?logo=github)](https://github.com/marcus-held/jenkins-library-subfolder-loader)

</div>

---

## Table of contents

- [Overview](#overview)
- [Build & local development](#build--local-development)
- [Repository layout](#repository-layout)
- [Contributing & conventions](#contributing--conventions)
- [Glossary](#glossary)
- [Ownership & support](#ownership--support)

---

## Overview

Jenkins normally loads a **shared library** from its own SCM source, so a repo can't simply use pipeline
helpers that live next to its code. `loadLocalLibrary` closes that gap. It checks out the build's own repo
on a node, turns one subfolder (for example `jenkins/library`) into a throwaway local git repo, and loads
that repo as the library **`local-lib@master`**. After the call, the steps in that subfolder's `vars/` are
available to the rest of the Jenkinsfile.

As of 2026-10-29 it is registered as the global library `subfolder_library_loader`, default version
`master`, in the MLP Jenkins config
([`Glovo/jenkins-configuration` › `mlp-jenkins-prod/global/jenkins.yaml`](https://github.com/Glovo/jenkins-configuration/blob/master/projects/configs/mlp/mlp-jenkins-prod/global/jenkins.yaml)).
Pipelines in [`Glovo/machine-learning-platform`](https://github.com/Glovo/machine-learning-platform) call it.

| | |
|---|---|
| **Language / build** | Groovy global step in [`vars/`](vars); Gradle [5.2.1](gradle/wrapper/gradle-wrapper.properties) for compile and IDE support only |
| **Runtime** | Jenkins pipeline; loaded from git by the Jenkins global-library config, no artifact |
| **Upstream** | [marcus-held/jenkins-library-subfolder-loader](https://github.com/marcus-held/jenkins-library-subfolder-loader) (MIT, [`LICENCE`](LICENCE)) |
| **Owner** | None declared (no `CODEOWNERS`, no `catalog-info.yaml`). See [Ownership & support](#ownership--support) |

### Using the step

A real call from a consuming Jenkinsfile:

```groovy
@Library('subfolder_library_loader') _      // the global-library name configured in that Jenkins
loadLocalLibrary scm, "jenkins/library"     // load ./jenkins/library from this repo as local-lib@master
```

Signature: `loadLocalLibrary(GitSCM scm, String libraryPath, String masterNode = 'master')`.

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `scm` | `hudson.plugins.git.GitSCM` | — | The SCM to check out; pass the pipeline's `scm` |
| `libraryPath` | `String` | — | Folder, relative to the repo root, that holds the library |
| `masterNode` | `String` | `'master'` | Label of the node the step runs on |

What [`loadLocalLibrary.groovy`](vars/loadLocalLibrary.groovy) does, in order:

1. Allocates a node with the label `masterNode`.
2. Runs `checkout scm`. If that throws, it wipes the workspace with `deleteDir()` and checks out again.
3. In `./<libraryPath>` it deletes any `.git`, then runs `git init`, `git add --all` and `git commit -m init`.
4. Calls `library identifier: 'local-lib@master'` with a `GitSCMSource` whose remote is the absolute path of
   that folder, with `changelog: false`.
5. Leaves the workspace in place. Upstream deleted it here; the Glovo change
   ([`fd7f9ab`](https://github.com/Glovo/jenkins-library-subfolder-loader/commit/fd7f9aba012f4d71980813a3adf4035b3200bbe7),
   2020-12-02) keeps it and only cleans on a failed checkout.

> [!NOTE]
> The step runs shell commands with `sh`, so the node must be a Unix agent with the **`git` CLI** on its
> path. The library identifier `local-lib@master` is hard-coded, so the subfolder is always loaded under
> that name.

**Further reading** (from the original author):
- [How to load a shared library from a subfolder in Jenkins](https://web.archive.org/web/20211023152650/https://code-held.com/2020/01/22/jenkins-local-shared-library/): the blog post that explains how the library works (archived copy).
- [Example project](https://github.com/marcus-held/jenkins-library-subfolder-loader-example): shows how to use the library.

---

## Build & local development

> [!NOTE]
> **Prerequisites:** a JDK that Gradle 5.2.1 runs on (8 to 11). The code targets Java 1.8
> (`sourceCompatibility` in [`build.gradle`](build.gradle)). The Gradle wrapper is committed, so no local
> Gradle install is needed.

```bash
./gradlew build      # compile vars/ against the Jenkins classpath declared in build.gradle
```

The Gradle build only gives you compile checks and IDE completion. Nothing is published: Jenkins loads
[`vars/`](vars) from the git branch set in its global-library config.

### Run locally

There is no local runner and the repo has no tests. To try a change, push a branch and point a test job at it:

```groovy
@Library('subfolder_library_loader@<your-branch>') _   // works if the Jenkins library allows version overrides
loadLocalLibrary scm, "jenkins/library"
```

### Build logic

| Setting | Value | Source |
|---|---|---|
| Gradle plugin | `com.mkobit.jenkins.pipelines.shared-library` 0.10.1 | [`build.gradle`](build.gradle) |
| Jenkins core | 2.201 (`sharedLibrary.coreVersion`) | [`build.gradle`](build.gradle) |
| Repositories | `jcenter()`, `mavenLocal()`; plugins from `gradlePluginPortal()` | [`build.gradle`](build.gradle), [`settings.gradle`](settings.gradle) |
| Dependency tweaks | excludes `org.slf4j:slf4j-jdk14`; forces `com.ibm.icu:icu4j:65.1` | [`build.gradle`](build.gradle) |
| Project name | `local-shared-library` | [`settings.gradle`](settings.gradle) |

<details><summary><b>Jenkins plugin dependencies on the compile classpath (11)</b></summary>

| Group | Artifact | Version |
|---|---|---|
| `org.jenkins-ci.plugins` | `pipeline-build-step` | 2.8 |
| `org.6wind.jenkins` | `lockable-resources` | 2.5 |
| `org.eclipse.hudson.plugins` | `git` | 3.0.1 |
| `org.jenkinsci.plugins` | `pipeline-model-api` | 1.3.9 |
| `org.jenkinsci.plugins` | `pipeline-model-declarative-agent` | 1.1.1 |
| `org.jenkinsci.plugins` | `pipeline-model-definition` | 1.3.9 |
| `org.jenkinsci.plugins` | `pipeline-model-extensions` | 1.3.9 |
| `org.jenkins-ci.plugins.workflow` | `workflow-step-api` | 2.20 |
| `org.jenkins-ci.plugins.workflow` | `workflow-cps` | 2.74 |
| `com.cloudbees` | `groovy-cps` | 1.31 |
| `org.jenkins-ci` | `symbol-annotation` | 1.17 |

</details>

---

## Repository layout

```text
.
├── vars/
│   └── loadLocalLibrary.groovy   ⭐ the loadLocalLibrary step: the only runtime code
├── build.gradle                  # shared-library Gradle plugin, Jenkins core and plugin versions
├── settings.gradle               # project name, plugin repository
├── gradle/wrapper/               # Gradle 5.2.1 wrapper
├── gradlew, gradlew.bat          # wrapper scripts
├── LICENCE                       # MIT, from upstream
└── .gitignore                    # ignores .gradle and build (generated)
```

---

## Contributing & conventions

> [!WARNING]
> **`master` is live.** The MLP Jenkins loads this library at its default version `master`, and the repo
> has no CI or tests. A merge reaches consuming pipelines on their next run, so try changes on a branch
> first (see [Run locally](#run-locally)).

1. **Keep the signature** `loadLocalLibrary(GitSCM scm, String libraryPath, String masterNode = 'master')`
   backward compatible. Consumers call it with positional arguments.
2. **Keep the identifier** `local-lib@master` unless you update consumers too: it is the name the loaded
   library gets.
3. **Keep the MIT notice** in [`LICENCE`](LICENCE). This is a fork, and the licence requires it.
4. **Know the fork delta.** Glovo's only change is
   [`fd7f9ab`](https://github.com/Glovo/jenkins-library-subfolder-loader/commit/fd7f9aba012f4d71980813a3adf4035b3200bbe7):
   don't delete the workspace after loading; clean it only if the checkout fails. On 2026-10-29 the fork was
   1 commit ahead of and 0 behind `marcus-held:master`.

There is no `CODEOWNERS` file, so GitHub requests no team review automatically.

---

## Glossary

| Term | Meaning |
|---|---|
| Shared library | Groovy code that Jenkins loads into pipelines from an SCM repo |
| Global library | A shared library registered in the Jenkins system config under a name, used with `@Library('<name>')` |
| `vars/` step | A `vars/<name>.groovy` file with a `call()` method; pipelines invoke it as `<name>` |
| `scm` | The pipeline variable that holds the SCM config the Jenkinsfile itself came from |
| `GitSCMSource` / `modernSCM` | The Jenkins retriever that loads a library from a git remote; here the remote is a local path |
| Workspace | The node directory where the job checks out the repo |
| Node label | The name that picks which Jenkins node runs the block; here `masterNode` |

---

## Ownership & support

- **Team:** none declared. The repo has no `CODEOWNERS` and no `catalog-info.yaml`.
- **Main consumer:** pipelines in [`Glovo/machine-learning-platform`](https://github.com/Glovo/machine-learning-platform),
  whose `/jenkins/` folder is owned by [`machine-learning-platform`](https://github.com/orgs/Glovo/teams/machine-learning-platform)
  in that repo's `CODEOWNERS`.
- **Upstream:** [marcus-held/jenkins-library-subfolder-loader](https://github.com/marcus-held/jenkins-library-subfolder-loader).
