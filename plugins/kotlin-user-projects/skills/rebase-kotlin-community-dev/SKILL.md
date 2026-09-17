---
name: rebase-kotlin-community-dev
description: Rebase the kotlin-community/dev branch of a Kotlin quality-gate fork onto the latest main/master from the JetBrains origin remote, in a separate working branch. Checks Gradle configuration with a dry run, has the user run a TeamCity build, tracks that build, and moves the result into kotlin-community/dev when it's green. Preserves the quality-gate customizations (custom Kotlin version, custom Kotlin Maven repository, language/API version overrides, compatibility fixes). Use when asked to rebase, sync, refresh, or update a Kotlin user-project / quality-gate / KCT fork, or to bring kotlin-community/dev up to date with main or master.
---

# Rebase kotlin-community/dev onto origin main/master

## Context

This repository is a Kotlin quality-gate fork of an open-source project, hosted on the JetBrains remote `origin`. The `kotlin-community/dev` branch holds a stack of commits on top of the project's default branch (`main` or `master`). Those commits let CI supply the Kotlin version and other settings externally, typically through Gradle properties such as:

- `kotlin_version`: the Kotlin Gradle plugin and stdlib version under test
- `kotlin_repo_url`: the custom Maven repository with dev/EAP Kotlin builds, often wired through `exclusiveContent`
- `kotlin_language_version` / `kotlin_api_version`: language and API version overrides
- extra compiler flags, warning suppression, and compatibility bumps (Gradle, AGP, and so on), often prefixed `[QG]`, `[KUP]`, or `[Backport]`

The goal is to move that commit stack onto the latest `origin/<default>` in a working branch, keep every customization working, verify the result on TeamCity, and only then move it into `kotlin-community/dev`.

## Hard Rules

- Use only the `origin` remote. Don't search for, add, or fetch from the original open-source repository or any other remote.
- Never rebase `kotlin-community/dev` directly. Rebase a working branch, and move the result into `kotlin-community/dev` only after the TeamCity build is green.
- Never push `kotlin-community/dev`. At the end, suggest the push command and leave it to the user.
- Push the newly created working branch to `origin` before asking for the TeamCity build: TeamCity can only run a build for a branch that exists on the remote. Push it only as a new branch, without force.
- Don't run a local build or any real Gradle tasks. The only local Gradle check is the configuration dry run.
- Never downgrade dependencies or tooling from the new base to make a customization apply. If a QG commit bumped a version and the base now has a newer one, keep the base's version.
- Never drop the external Kotlin version or repository mechanism to make things pass.
- Don't commit machine-local values (`local.properties`, uncommented `kotlin_version=` / `kotlin_repo_url=` in `gradle.properties`, regenerated `Podfile.lock` / podspec noise) unless the original stack already committed them.
- Leave `kotlin-community/<version>` release branches alone.

## Step 1. Check Local Changes and Branch State

1. Run `git branch --show-current` and `git status --short`.
2. If the working tree has changes, show them and ask whether to stash (`git stash push -u -m "pre-rebase <date>"`), commit, or discard them. Local edits to `gradle.properties` that set `kotlin_version` / `kotlin_repo_url` are usually test leftovers, but still ask. Ignore untracked build output such as `.kotlin/`, `build/`, and `.gradle/`.
3. If the current branch isn't `kotlin-community/dev`, tell the user and ask before switching.
4. Run `git fetch --prune origin`.
5. Make sure local `kotlin-community/dev` matches `origin/kotlin-community/dev`:
   ```bash
   git rev-list --left-right --count kotlin-community/dev...origin/kotlin-community/dev
   ```
   - `0 0`: up to date.
   - Behind only: fast-forward with `git merge --ff-only origin/kotlin-community/dev`.
   - Ahead or diverged: show the commits and ask the user how to proceed.
6. Find the default branch with `git remote show origin | sed -n 's/.*HEAD branch: //p'`. It's usually `main` or `master`. Below, `<default>` means that branch.
7. Read the repo conventions: `AGENTS.md`, `CLAUDE.md`, `README`, and the CI config (`.teamcity/`, `.github/`, `.space.kts`) if present.

## Step 2. Create the Working Branch

```bash
REBASE_BRANCH=kotlin-community/rebase-$(date +%Y%m%d)
git checkout -b $REBASE_BRANCH kotlin-community/dev
OLD_BASE=$(git merge-base kotlin-community/dev origin/<default>)
```

If a branch with that name already exists locally or on `origin`, ask the user whether to reuse it, delete it, or pick another name.

Before rebasing, note what must survive:

```bash
git log --reverse --format='%h %s' $OLD_BASE..kotlin-community/dev
git diff $OLD_BASE kotlin-community/dev -- '*.gradle*' '*.properties' 'gradle/' 'buildSrc/' 'build-logic/'
git diff --stat $OLD_BASE origin/<default> -- '*.gradle*' '*.properties' 'gradle/' 'settings*' 'buildSrc/' 'build-logic/'
```

Write a short list of the customizations: where `kotlin_version`, `kotlin_repo_url`, and the LV/AV properties are read, how the repositories are filtered, which versions were bumped, and which workarounds were added. If the base had significant migrations (a version catalog, a convention plugin, `buildSrc` to `build-logic`, a Kotlin or Gradle upgrade, a removed module), tell the user before rebasing.

## Step 3. Rebase

```bash
git rebase origin/<default>
# if the old base isn't an ancestor of origin/<default> (history was rewritten):
git rebase --onto origin/<default> $OLD_BASE $REBASE_BRANCH
```

Use a non-interactive rebase. Interactive flags aren't supported here.

### Resolving Conflicts

For each stop:

1. Run `git status` and `git show --stat REBASE_HEAD` to see which QG commit is being applied and why it exists.
2. Resolve by taking the new base's code as the starting point and reapplying the commit's intent on top. Don't restore old base code.
   - **Version bumps** (Gradle wrapper, AGP, Ktor, Hilt, Node, and so on): take the newer of the base's and the fork's version. If the base is already newer or equal, the hunk becomes a no-op.
   - **Kotlin version wiring**: keep the base's default Kotlin version as the fallback, but keep the external `kotlin_version` override, whether it's in a version catalog, `pluginManagement`, `resolutionStrategy`, or a `plugins {}` block.
   - **Repository wiring**: keep the base's new repositories and keep the custom `kotlin_repo_url` repository with its `exclusiveContent` / `includeVersionByRegex("org.jetbrains.kotlin*", ".*", kotlinVersion)` filter.
   - **Deprecation migrations** (`kotlinOptions` to `compilerOptions`, `android()` to `androidTarget()`, and so on): if the base already migrated, keep the base's version and reapply only the QG-specific additions, such as LV/AV and extra compiler flags.
   - **Lock files and generated files** (`Podfile.lock`, `*.podspec`, `kotlin-js-store/yarn.lock`, `gradle/verification-metadata.xml`): prefer the base's copy. Don't regenerate them locally.
3. If a commit becomes empty, run `git rebase --skip` and record it as "dropped: already in base".
4. If the base already has a commit's intent in a different form, it's fine to drop that commit. Record why.
5. Run `git add <files>`, then `GIT_EDITOR=true git rebase --continue`.
6. Use `git rebase --abort` only after asking the user. `kotlin-community/dev` still holds the original state either way.

Keep commit messages and authorship unchanged. When a commit's content had to change meaningfully, keep the subject and mention the adaptation in the report, not in the message.

## Step 4. No Changes: Report and Exit

If the rebase changed nothing, the working branch points to the same commit as `kotlin-community/dev` (`git rev-parse $REBASE_BRANCH` equals `git rev-parse kotlin-community/dev`). This happens when `origin/<default>` has no new commits. In that case:

1. Tell the user that `kotlin-community/dev` is already based on the latest `origin/<default>`. Include the SHA and date of `origin/<default>`.
2. Switch back with `git checkout kotlin-community/dev`, delete the working branch with `git branch -D $REBASE_BRANCH`, and restore any stash from step 1 if the user wants it.
3. Stop. Don't continue to the next steps.

## Step 5. Changes Exist: Review the Result

If the working branch moved, continue:

1. Compare the old and new stacks commit by commit:
   ```bash
   git range-diff $OLD_BASE..kotlin-community/dev origin/<default>..$REBASE_BRANCH
   ```
   Check every dropped or changed commit against the list of customizations from step 2.
2. Check that the customizations still exist:
   ```bash
   git grep -nE 'kotlin_repo_url|kotlin_version|kotlin_language_version|kotlin_api_version' -- ':!*.md'
   ```
3. Run `git status` and make sure the tree is clean.

## Step 6. Gradle Configuration Dry Run

Check only that Gradle configures successfully with the Kotlin version under test. `--dry-run` skips every task, so `help` only satisfies the need for a task name:

```bash
./gradlew --dry-run help -Pkotlin_version=<ver> -Pkotlin_repo_url=https://redirector.kotlinlang.org/maven/dev
```

- For `<ver>`, use the Kotlin version from the latest `kotlin-community/<version>` branch, or ask the user.
- Pass `kotlin_language_version` / `kotlin_api_version` only if the user asks or CI passes them.
- If the build uses configuration on demand, add `-Dorg.gradle.configureondemand=false` so the whole build is configured.

If configuration fails, assume the rebase caused it until the logs prove otherwise. Fix a rebase regression, such as a lost customization or a bad conflict resolution, by amending the relevant commit: use `git commit --fixup <sha>`, then `GIT_SEQUENCE_EDITOR=true git rebase -i --autosquash origin/<default>`. Rerun the dry run afterward. For a failure that doesn't come from the rebase, report it and ask the user how to proceed.

## Step 7. Push the Working Branch and Ask for a TeamCity Build

1. Push the working branch to `origin` before asking for a build. TeamCity can only build a branch that exists on the remote, so without this push there's nothing to run. It's a new branch, so push it without force:
   ```bash
   git push -u origin $REBASE_BRANCH
   git ls-remote --heads origin $REBASE_BRANCH   # confirm the branch is on origin
   ```
   If the push fails (for example, because a branch with that name already exists on `origin`), don't force it. Report the error and ask the user how to proceed.
2. Tell the user that the rebase is done and that `$REBASE_BRANCH` is pushed to `origin`. Summarize the old base and new base (SHA and date), the number of new base commits, the commits you kept, adapted, or dropped, the conflicts you resolved, and the dry-run result.
3. Ask the user to start the TeamCity build for `$REBASE_BRANCH` and share the link to it. Then wait for the link.

## Step 8. User Starts the Build

The user starts the build on TeamCity and shares the build URL. Take the build ID from the URL (for example, `.../buildConfiguration/<job>/<buildId>` or `...viewLog.html?buildId=<buildId>`). If the URL is ambiguous, confirm the ID with the user.

## Step 9. Track the Build

Load the `teamcity-cli` skill and follow its guidance. Don't guess flags. In short:

```bash
teamcity auth status
teamcity run view <buildId>
teamcity run watch <buildId> --timeout 3h --quiet   # run in the background
```

Run the watch in the background so the session isn't blocked. When it finishes, check the final status with `teamcity run view <buildId>`. If this is a composite build or a build chain, check every child build with `teamcity run tree <buildId>`.

If the build fails:

1. Find the root cause: `teamcity run tree <buildId>`, then `teamcity run log <id> --failed --raw` on the deepest failed build, and `teamcity run tests <id> --failed`.
2. Classify each failure as one of these:
   - a rebase regression
   - an incompatibility between the new base and the tested Kotlin version
   - a breakage in the base that doesn't depend on Kotlin
   - an infrastructure issue
3. Report the failures to the user and propose fixes. Leave `kotlin-community/dev` unchanged.
4. If the user asks for fixes, commit them on the working branch. Rebase regressions go in as fixup commits; new Kotlin-compatibility fixes go in as `[QG] ...` commits. Rerun the dry run, then ask before pushing the updated working branch; if the history was rewritten, push with `--force-with-lease`. Then return to step 7 for a new build.

## Step 10. Green Build: Move the Result into kotlin-community/dev

When the build, including every child build, is green:

1. Tell the user that the build passed and include the link.
2. Back up the current `kotlin-community/dev`, then move it to the working branch locally:
   ```bash
   OLD_DEV=$(git rev-parse kotlin-community/dev)
   git branch backup/kotlin-community-dev-$(date +%Y%m%d) $OLD_DEV
   git checkout kotlin-community/dev
   git reset --hard $REBASE_BRANCH
   ```
3. Restore any stash from step 1 if the user wants it (`git stash pop`).
4. Suggest the push, but don't run it. It rewrites the remote history of `kotlin-community/dev`:
   ```bash
   git push --force-with-lease=kotlin-community/dev:<OLD_DEV sha> origin kotlin-community/dev
   ```
5. Also mention the cleanup the user can do after pushing: delete the local working branch, `origin/$REBASE_BRANCH` (`git push origin --delete $REBASE_BRANCH`), and the backup branch.

## Final Report

Report:

- Old base and new base on `origin/<default>` (short SHA and date), and how many base commits were pulled in.
- The working branch name, and whether it was pushed.
- The commit stack before and after, marking each commit as kept, adapted (with how), or dropped (with why).
- Conflicts resolved, by file.
- The dry-run command and its result.
- The TeamCity build link and its status.
- Whether `kotlin-community/dev` was updated locally, the backup branch name, and the suggested push command.
