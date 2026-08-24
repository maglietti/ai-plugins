# Documentation

Guides for the `ai-plugins` repository, split by what you are trying to do. For installing the plugins, the root [README.md](../README.md) is the reference.

## Using the plugins

Start here if you have installed a plugin and want to get work out of it.

- **[Using the MariaDB skills](using-the-skills.md)** explains why you never call a skill directly, what the 75 skills in the `dev` plugin cover, how to phrase a task so the right one loads, and how to prove any of it is working.

## Recipes

Multi-step workflows, each backed by an end-to-end test that drives a real Claude Code CLI.

- **[Expose a schema over REST](recipes/rest-service.md)** builds a schema, runs it on a throwaway sandbox instance, and puts a REST API in front of it. One prompt, roughly five minutes.
- **[Version a schema with MSM](recipes/schema-lifecycle.md)** carries a schema through two releases and the migration between them, without touching a database.

## Working on the repository

- **[Getting started](getting-started.md)** covers the contributor side, including how skills are vendored, what the four test tiers do, how the two version numbers move, and the traps that have already cost time.

## Reference

- [additional-skills/README.md](../additional-skills/README.md) records where every vendored skill came from and which license it carries.
- [.claude/PROJECT_CONTEXT.md](../.claude/PROJECT_CONTEXT.md) is the working session log, with architecture decisions and their reasoning.
- Every plugin directory carries its own `README.md` and `CHANGELOG.md`, and every test suite has a `README.md` covering its tiers.
