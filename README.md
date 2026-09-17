# Agent plugins

A [Codex](https://github.com/openai/codex) plugin marketplace.

## kotlin-user-projects

Skills for maintaining Kotlin user-project forks.

A user-project fork is a fork of an open-source project that serves as a quality gate for new Kotlin versions. The fork keeps a `kotlin-community/dev` branch on top of the project's `main`/`master`. That branch lets CI supply the Kotlin version, a custom Kotlin Maven repository, and language/API versions through Gradle properties such as `kotlin_version`, `kotlin_repo_url`, `kotlin_language_version`, and `kotlin_api_version`.

### Skills

The `kotlin-user-projects` plugin contains these skills:

| Skill | What it does |
| --- | --- |
| `rebase-kotlin-community-dev` | Rebases `kotlin-community/dev` onto the latest `origin` `main`/`master` in a working branch and checks Gradle configuration with a dry run. Then it pushes the working branch so you can start a TeamCity build, tracks that build, and, when it's green, moves the result into `kotlin-community/dev` locally. It suggests the push but doesn't run it. |
| `update-kotlin-repo-exclusive-content` | Declares the custom Kotlin Maven repository (`kotlin_repo_url`) through Gradle `exclusiveContent`, so it serves only Kotlin artifacts of the supplied Kotlin version. |

### Requirements

- Codex CLI with plugin support (`codex plugin`).
- For `rebase-kotlin-community-dev`: the [TeamCity CLI](https://github.com/JetBrains/teamcity-cli) (`teamcity`) with its `teamcity-cli` skill, authenticated against your TeamCity server.

## Install

```bash
codex plugin marketplace add git@github.com:atyrin/agent-plugins.git
codex plugin add kotlin-user-projects@agent-plugins
```

To update:

```bash
codex plugin marketplace upgrade
```

To install from a local checkout instead:

```bash
codex plugin marketplace add /path/to/agent-plugins
codex plugin add kotlin-user-projects@agent-plugins
```

## Usage

In a fork's repository, ask Codex, for example:

- "Rebase kotlin-community/dev onto master."
- "Wire kotlin_repo_url through exclusiveContent."

## Layout

```
.agents/plugins/marketplace.json            # marketplace manifest
plugins/kotlin-user-projects/
  .codex-plugin/plugin.json                 # plugin manifest
  skills/<skill-name>/SKILL.md              # skill instructions
  skills/<skill-name>/agents/openai.yaml    # skill UI metadata
```

To add a skill, create `plugins/kotlin-user-projects/skills/<skill-name>/SKILL.md` and bump `version` in `plugin.json`.
