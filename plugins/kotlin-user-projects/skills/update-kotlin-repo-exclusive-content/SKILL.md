---
name: update-kotlin-repo-exclusive-content
description: Update forked Kotlin user-project repositories so a custom Kotlin Maven repository supplied through Gradle properties is declared with Gradle exclusiveContent filtering. Use when Codex is working in an open-source project fork used to test new Kotlin versions, especially branches such as kotlin-community/dev, and needs to replace kotlin_repo_url?.also { maven(it) } or similar custom repository wiring in build scripts, settings.gradle.kts, buildSrc, convention plugins, pluginManagement, or included builds without downgrading Kotlin.
---

# Update Kotlin Repo Exclusive Content

## Goal

Update a Kotlin user-project fork so the custom Kotlin Maven repository can only serve Kotlin artifacts for the externally supplied Kotlin version. Do not downgrade Kotlin or change Kotlin version selection to make the build pass.

## Inputs to Identify

Inspect the project before editing:

- Locate Gradle properties and aliases for the custom Kotlin repository URL, commonly `kotlin_repo_url`, `kotlinRepoUrl`, `kotlin_repo_url?.also { maven(it) }`, or `maven(kotlin_repo_url)`.
- Locate the externally supplied Kotlin version value, commonly `kotlin_version`, `kotlinVersion`, `kotlin_language_version`, or a project-specific wrapper around those properties.
- Compare the current branch with `master` or `main` when available to find user-project customizations: `git diff master...HEAD` or `git diff main...HEAD`.
- Search all Gradle entry points, not only the root build script: `settings.gradle.kts`, `settings.gradle`, `build.gradle.kts`, `build.gradle`, `buildSrc`, `gradle/`, convention plugins, precompiled script plugins, included builds, and plugin-management blocks.

## Branch Preparation

Before editing project files:

1. Check the current branch with `git branch --show-current`.
2. Run `git pull --ff-only` to update local refs and make newly published branches visible. If the pull fails because of local changes or divergent history, stop and ask the user how to proceed.
3. If the current branch is not `kotlin-community/dev`, tell the user the current branch and ask for confirmation before switching to `kotlin-community/dev`.
4. After confirmation, check out `kotlin-community/dev`. If only `origin/kotlin-community/dev` exists, create the local tracking branch from the remote branch.
5. Run `git pull --ff-only` again after switching to `kotlin-community/dev`.

## Required Repository Shape

Replace direct custom Maven repository insertion with `exclusiveContent` wherever the custom Kotlin repository is added to dependency resolution repositories.

For Kotlin DSL, prefer this shape, adapting variable names to the project:

```kotlin
if (!kotlin_repo_url.isNullOrEmpty() && !kotlinVersion.isNullOrEmpty()) {
    exclusiveContent {
        forRepository {
            maven(kotlin_repo_url)
        }
        filter {
            includeVersionByRegex("org.jetbrains.kotlin*", ".*", kotlinVersion)
        }
    }
}
```

For Groovy DSL, use the equivalent Gradle API:

```groovy
if (kotlinRepoUrl && kotlinVersion) {
    exclusiveContent {
        forRepository {
            maven { url = uri(kotlinRepoUrl) }
        }
        filter {
            includeVersionByRegex('org.jetbrains.kotlin*', '.*', kotlinVersion)
        }
    }
}
```

Keep the project's existing property validation when it exists. If the project already fails early when `kotlin_repo_url` or the Kotlin version is missing, do not weaken that validation. If guards are needed locally, guard both the repository URL and Kotlin version.

## Workflow

1. Complete branch preparation before editing.
2. Read the repository conventions first: check `AGENTS.md`, build logic, and existing property helpers before editing.
3. Identify every place the custom Kotlin Maven repository is wired into Gradle repositories.
4. Preserve normal repositories such as Maven Central, Google, Gradle Plugin Portal, and project-specific repositories.
5. Convert only the custom Kotlin repository wiring to `exclusiveContent`.
6. Use `includeVersionByRegex("org.jetbrains.kotlin*", ".*", kotlinVersion)` so only Kotlin modules at the supplied Kotlin version can resolve from the custom repository.
7. Do not add broad filters such as `includeGroupByRegex(".*")`, and do not allow non-Kotlin dependencies to resolve from the custom Kotlin repository.
8. Handle plugin resolution separately if the project adds the custom repository in `pluginManagement.repositories`; use `exclusiveContent` there too when supported by the Gradle version in the project, but first check whether the build still adds repositories through `buildscript.repositories`.
9. If `pluginManagement` uses exclusive content and the build also declares `buildscript.repositories`, either centralize the needed repositories in settings or leave plugin-management filtering non-exclusive and report why.
10. Keep edits tightly scoped to repository declaration and helper code needed for that declaration.

## Validation

Validate the actual project change, not only syntax:

```bash
./gradlew resolveDependencies -Pkotlin_version=2.4.20-Beta1-2 -Pkotlin_repo_url=https://redirector.kotlinlang.org/maven/dev
```

Adapt the Kotlin version, repository URL, and property names to the target project or task instructions. Include all additional gate properties the project requires, such as `kotlin_language_version` or `kotlin_api_version`; do not remove or downgrade Kotlin properties to make validation pass.

Do not treat a failed validation command as an infrastructure problem by default. Assume with roughly 99% probability that the failure is caused by the changed repository-filtering code unless the logs contain clear independent evidence of an infrastructure outage, unavailable network, or broken remote repository.

If `resolveDependencies` is unavailable, inspect the project for the nearest equivalent dependency-resolution task and explain the substitution. If dependency resolution fails, classify the failure as one of:

- a repository-filtering issue caused by the change,
- a Kotlin artifact availability issue for the requested version,
- an unrelated project failure.

## Commit And Cherry-Pick

After the change is validated:

1. Review `git diff` and ensure only the intended repository-filtering changes are included.
2. Commit the change on the current branch.
3. Cherry-pick the commit to release branches when they exist and the user requested propagation, especially:
   - `kotlin-community/2.4.0`
   - `kotlin-community/2.4.20-Beta1`
4. Resolve cherry-pick conflicts by preserving the same exclusive repository behavior in each branch, then run the same `resolveDependencies` validation on each changed branch.
5. Before pushing any branch, explicitly warn the user that the next step will push commits to the remote and ask for confirmation.
6. After confirmation, push the current branch and every release branch that received the cherry-pick.

Do not force-push, rewrite unrelated history, or revert user changes unless explicitly requested.

## Reporting

Report:

- Files changed and every repository block updated.
- The exact Gradle command(s) run, including the supplied repository and Kotlin version properties if they are not sensitive.
- The commit hash created, cherry-pick results per branch, and push status per branch.
- Any places intentionally left unchanged and why.
- Any remaining failure as a repository-filtering issue, Kotlin artifact availability issue, or unrelated project failure.
