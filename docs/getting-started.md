# Getting started with `ai-plugins`

This guide gets you working *on* the `ai-plugins` repository. If you only want to install the plugins and use them, read the root [README.md](../README.md) instead. Everything below is what you need before you change anything here.

## What you are working on

The repository packages MariaDB agent skills, which are Markdown files named `SKILL.md` that an AI coding agent loads when a task matches their description. It pairs those skills with the native [`mariadb-shell`](https://github.com/mariadb-corporation/mariadb-shell) server for the Model Context Protocol (MCP), the wire protocol an agent uses to reach a live database. You ship that pair as an installable plugin for four coding agents, which this repository calls harnesses.

| Harness | Directory | Variants shipped |
| ------- | --------- | ---------------- |
| Claude Code | `claude/` | `dev`, `sql`, `contributor` |
| Codex | `codex/` | `dev`, `sql`, `contributor` |
| OpenCode | `opencode/` | `dev`, `sql`, `contributor` |
| Pi (pi.dev) | `pi/` | `dev` only |

That comes to ten plugin directories. The three variants differ only in what they carry.

| Variant | Skills on disk today | MCP server | Skill sources |
| ------- | -------------------- | ---------- | ------------- |
| `dev` | 75 | yes | `mariadb-docs` agent-skills plus all of `additional-skills/` |
| `sql` | 47 | yes | the SQL-focused upstream layers plus `additional-skills/sql/` |
| `contributor` | 1 (`create-shell-plugin`) | no | the private `mariadb-shell` repository's `.claude/skills/` |

Every skill is written against MariaDB 11.8 LTS. Two version numbers both sit at `26.8.0` right now, and they move independently of each other. One is the plugin package version. The other is the minimum `mariadb-shell` version a plugin will accept. [Change a version](#change-a-version) covers which script owns which.

## Four things to know before you edit anything

**Skills are vendored, never authored in place.** Every `<harness>/<variant>-plugin/skills/` tree is generated output. The real sources are the upstream `mariadb-corporation/mariadb-docs` repository and this repository's own `additional-skills/` folder. Edit a vendored copy and it works locally, right up until the next sync silently reverts it. Make the change in the source, then re-run the `scripts/sync-skills.sh` script.

**Every plugin uses the same flat skill layout,** `skills/<skill-name>/SKILL.md`, no matter how upstream groups the skills. OpenCode discovers skills only one directory deep, and pi's loader fails outright on any directory under `skills/` that has no `SKILL.md` in it. So keep `skills/` holding nothing but skill directories, the `.skills-manifest.json` manifest, and the `skills-source.json` provenance file.

**The MCP server installs itself on first use.** Each plugin ships the `mariadb-mcp-launcher.sh` script rather than a binary, with a `.cmd` twin for native Windows. The launcher looks for a usable `mariadb-shell` in four places, in order: the `$MARIADB_SHELL_BIN` override, a copy on your `PATH` at or above `MARIADB_SHELL_VERSION`, an existing install under `~/.local/bin`, and finally the shell's own `install.sh` or `install.ps1`, which it fetches and runs only when the first three come up short. It then execs whatever it found as an MCP server over stdio. Note that stdout *is* the JSON-RPC transport, so every message the launcher and the installer emit has to go to stderr.

**The launcher is one file copied fourteen times.** Seven copies of the `.sh` version and seven of the `.cmd` version sit in the plugin directories, byte for byte identical. Edit the copy in `claude/dev-plugin/scripts/`, `cp` it over the rest, then confirm with `shasum`. Keep the `.cmd` copies CRLF and ASCII. The `.gitattributes` file pins that for you, because an LF batch file mis-parses the `goto` labels and `^` line continuations the launcher leans on.

## What to install first

For static work on skills, docs, and structural tests, you need only Python 3.12 and `pytest`. Everything else in the table below buys you a specific capability.

| Tool | What it unlocks |
| ---- | --------------- |
| `jq` and `curl` | the `scripts/sync-skills.sh` script |
| `gh` authenticated, or `GH_TOKEN` | syncing the `contributor` plugins and installing `mariadb-shell`, both from private repositories |
| `mariadb-shell` on your `PATH` | the `run_tests.py` runner, the `db` tier, the `e2e` tiers |
| Docker | running the `db` tier against a container instead of a sandbox instance |
| An authenticated harness CLI (`claude`, `codex`, `pi`) | the `e2e` tiers |
| `ANTHROPIC_API_KEY` or `OPENAI_API_KEY` | the `eval` tier |

Watch out for one trap while the `mariadb-shell` repository stays private. Anonymous requests to it answer 404 rather than 401, so a missing token looks exactly like a missing repository. When an install or a sync fails with "not found", check your token before you check anything else.

Do one more setup step per machine, because it is easy to miss. The MCP server starts out allowed to reach nothing at all, so run `mariadb-shell -- mcp setup` once to give it an allow-list.

## Where things live

```text
ai-plugins/
├── .claude-plugin/marketplace.json    Claude Code marketplace entry
├── .agents/plugins/marketplace.json   Codex marketplace entry (Codex reads this one)
├── additional-skills/                 repo-local skill sources
│   ├── sql/                            1 skill, vendored into dev and sql
│   ├── rest/                           5 MariaDB REST Service skills, dev only
│   └── schema-management/              5 schema-lifecycle skills, dev only
├── claude/  codex/  opencode/  pi/    one directory per harness
│   ├── <variant>-plugin/              generated plugin, holding skills/, scripts/, MCP config
│   └── dev-plugin-test[s]/            that harness's pytest suite
├── scripts/
│   ├── sync-skills.sh                 vendors skills into every plugin
│   ├── set-plugin-version.sh          the plugin package version
│   └── set-mariadb-shell-version.sh   the mariadb-shell minimum version
├── run_tests.py                       runs every suite under the mariadb-shell Python
├── pytest-coverage.ini, .coveragerc   shared test and coverage config
├── package.json                       the pi manifest, whose `pi` field points at pi/dev-plugin/
└── .github/workflows/                 one workflow per harness
```

One naming quirk will trip up your shell globs. The Claude and pi suites are `dev-plugin-tests` with an `s`, while the Codex and OpenCode suites drop it. The `run_tests.py` runner finds them all with the glob `*/dev-plugin-test*/conftest.py`, which is why both spellings have survived.

Treat pi as the odd harness. It has no marketplace file and no MCP support of its own. Any directory carrying a `package.json` with a `pi` field counts as a pi package, so the repo-root `package.json` acts as pi's manifest and the whole repository installs as a single package. MCP reaches pi through the `pi-mcp-adapter` community extension, which pi loads only as a package in its own right, so you install it separately.

## Install a plugin from your checkout

**Claude Code.** Add this directory as a marketplace, then install from it.

```text
/plugin marketplace add /Users/maglietti/Code/mariadb/ai-plugins
/plugin install dev@mariadb
```

**Codex.** Run those same two commands, then register the MCP server by hand. Codex 0.147 stores a plugin's `command` verbatim and expands no placeholder in it, which leaves a plugin unable to register a server that will actually start.

```sh
codex/dev-plugin/scripts/setup-codex-mcp.sh     # --remove to unregister
```

**OpenCode.** Merge the `mcp` block from `opencode/dev-plugin/opencode.json` into your own `opencode.json`. Then point the `MARIADB_DEV_PLUGIN` variable at the plugin directory and symlink the flat `skills/` tree into an OpenCode skills directory.

**Pi.** Install the adapter and this repository as two separate packages, then run the setup command that the extension registers for you.

```sh
pi install npm:pi-mcp-adapter
pi install .                    # from the repo root
```

```text
/mariadb-mcp-setup              # writes ~/.config/mcp/mcp.json
/mariadb-mcp-setup --project    # or ./.mcp.json for one project
```

## Change a skill

```sh
scripts/sync-skills.sh          # latest upstream default branch
scripts/sync-skills.sh <ref>    # a specific tag, branch, or commit
```

The `sync-skills.sh` script downloads the upstream `agent-skills/` tree once and copies the selected skills into each plugin's `skills/` directory. It rewrites the `.skills-manifest.json` manifest to flat paths while preserving each skill's layer grouping, then records the resolved commit SHA in every plugin's `skills-source.json`. Two variables near the top of the script decide which subfolders land where. `ADDITIONAL_SUBDIRS` lists this repository's own subfolders, and `SQL_INCLUDE_LAYERS` names the layers that the `sql` variant takes.

The `contributor` plugins come from a second source, the private `mariadb-shell` repository. Without `GH_TOKEN` or a working `gh auth token`, that step prints a warning and skips itself, leaving the `dev` and `sql` sync to succeed on its own.

Two habits are worth forming here. First, edit `additional-skills/` and skip the sync, and you ship stale text to every plugin with nothing to warn you. The manifest still matches disk, and the stale copy still parses cleanly. That one hid for four days before anyone caught it, and the `test_additional_skill_matches_its_source` static test now guards against a repeat. Second, remember that a sync at the upstream default branch head can drag in unrelated churn. Read `git diff --shortstat` and one sample diff before you conclude that a 432-file sync means 432 content revisions.

## Change a version

The two version numbers move independently, and each has a script that owns it. Resist editing the files by hand, because both scripts reach into more places than you will remember.

```sh
scripts/set-plugin-version.sh 26.9.0        # the plugin package version
scripts/set-mariadb-shell-version.sh 26.9.0 # the mariadb-shell minimum
```

The `set-plugin-version.sh` script rewrites the `version` field in every plugin manifest and the `Version **x.y.z**` line in every plugin README, across all four harnesses, plus the repo-root `package.json` that stands in for pi's missing per-plugin manifest. Its sibling, `set-mariadb-shell-version.sh`, rewrites `MARIADB_SHELL_VERSION` across the 35 files carrying it: the launchers, `.mcp.json`, `opencode.json`, `setup-pi-mcp.sh`, `setup-codex-mcp.{sh,cmd}`, and the READMEs' prose. Keep that prose line on one line, or the substitution stops matching it.

Check that second script whenever a file gains a version site. Its reach has already lagged behind three times, once when the pi harness arrived, once while the disabled launchers existed, and once when the setup scripts gained a default of their own.

Read `MARIADB_SHELL_VERSION` as a floor for accepting a binary you already have, never as a trigger to go looking for a newer one. A machine sitting on `26.8.0` keeps using it long after `26.9.0` ships. A download happens only when nothing on the machine clears the floor.

## Run the tests

All four suites share the same four tiers and the same markers.

| Tier | Marker | Needs | Checks |
| ---- | ------ | ----- | ------ |
| 1 | `static` | nothing | frontmatter, manifest against disk, cross-references, SQL fences, the statement-skill contract |
| 2 | `db` | `mariadb-shell` plus a server binary | the skills' recommended DDL runs on a live server with the documented effect |
| 3 | `eval` | an LLM API key | the skill steers the model toward the MariaDB-preferred form |
| 4 | `e2e` | the harness CLI, authenticated | drives the real CLI with the plugin loaded, then asserts side effects |

Tiers 1 and 2 behave identically everywhere. Tiers 3 and 4 are opt-in, deselected by default, and vary by harness. Claude and Codex each get an `eval` tier plus two `e2e` workflows, one for the MariaDB REST Service and one for the MariaDB Schema Management (MSM) lifecycle. OpenCode has the `eval` tier only. Pi runs `e2e` alone, since it has no SDK of its own to prompt, which leaves driving the `pi` binary as its behavioral test.

Reach for the repo-root `run_tests.py` runner first. It executes every suite with the Python that ships inside `mariadb-shell` by way of `mariadb-shell --pym pytest`, so your tests run against the same interpreter and packages as the MCP server they exercise. It also appends coverage across the runs into a single report.

```sh
./run_tests.py                  # every suite, default tiers (static and db)
./run_tests.py claude           # one suite, named after its top-level directory
./run_tests.py -m static        # one tier, all suites
./run_tests.py claude -m e2e    # the opt-in end-to-end tier
./run_tests.py -k manifest      # only tests matching a pattern
./run_tests.py -- --lf -x       # anything after `--` goes to pytest

npm test                        # the same, plus test:static, test:db, test:eval, test:e2e
```

The runner takes its binary from `--shell`, then `$MARIADB_SHELL`, then your `PATH`, and prepends that binary's directory to the `PATH` the suites see so the `e2e` tier's launcher resolves the same one. Pass `--no-install` to skip installing each suite's `requirements.txt`. Pass `--no-coverage` to drop the measurement, or `--userhome` to relocate the shell's user config home, which otherwise stays untouched.

To drive pytest directly instead, work inside a suite directory.

```sh
cd claude/dev-plugin-tests
pip install -r requirements.txt
pytest -m static
```

You need no server running before the `db` tier. It deploys a throwaway sandbox instance on a free port through the MCP server's `sandbox.*` tools and deletes it afterwards, with the shell's config home isolated to a temp directory for the duration. Set any of `MARIADB_HOST`, `MARIADB_PORT`, `MARIADB_USER`, or `MARIADB_PASSWORD` to point the tier at a server you already have, which is how CI and each suite's `docker-compose.yml` run it.

Expect these counts from the last full local run. The `static` tier passes 602 tests in the Claude, Codex, and OpenCode suites, and 611 in pi, where nine extra tests guard pi-specific wiring. The default tiers together come to 618 in those first three suites, and Claude's `e2e` tier adds 12 more. Reports land in `test-results/` and `htmlcov/`, both git-ignored.

## What CI runs

Four workflows sit in `.github/workflows/`, one per harness. Each runs the `static` tier on every push and pull request, then the `db` tier on pull requests against a `mariadb:11.8` service container. The three suites that have an `eval` tier run it nightly or on demand. The `e2e` tiers stay local, since they need an authenticated CLI and a resolvable `mariadb-shell`, and a GitHub runner offers neither.

CI still calls `pytest` per suite through `actions/setup-python` rather than going through `run_tests.py`, and that is deliberate. Runners have no `mariadb-shell` on them, which makes the shell-Python runner a local path only. So the per-suite `pyproject.toml` files stay authoritative for CI, while `pytest-coverage.ini` governs what you run at your desk. Keep the markers in those two places in sync by hand.

## Traps that have already cost time

- **Keep batch files CRLF, and put any license header after `@echo off`.** Command echo stays on until `@echo off` runs, so `rem` lines above it print to stdout, which for a launcher is the MCP transport.
- **Codex spawns MCP servers with a filtered environment.** Exported variables such as `MARIADB_SHELL_USER_CONFIG_HOME` never reach the server, so it falls back to the real `~/.mariadb-shell` and whatever allow-list lives there.
- **Pass `stdin=subprocess.DEVNULL` whenever `subprocess.run` drives `codex exec`.** Codex reads a piped stdin as extra prompt input and blocks until the timeout, leaving an empty event stream as your only symptom.
- **Pi ignores project-local packages unless the run passes `--approve`.** The `pi install -l` command reports success either way, and without approval your skills are simply absent, with no error to tell you.
- **Seed the MCP allow-list with both macOS path spellings.** `$TMPDIR` yields `/var/folders/...` while `Path.resolve()` yields `/private/var/folders/...`, and the path guard compares the two as strings.
- **Do not assert model obedience and call it wiring.** Two tests have gone red demanding a fixed output shape while the plugin underneath worked fine. Assert on the side effects the plugin produces rather than on whatever words the model picks.
- **Test a resolution chain by measuring the branch, not the outcome.** A probe keyed on "did the shell launch" reported success for every version, because the install-failure fallback launches a rejected binary too. Assert on the launcher's own stderr line instead.
- **Piping a test run through `tee` reports tee's exit code rather than pytest's.** Read the pytest summary line.

## Where to look next

- The root [README.md](../README.md) is the install-facing document, and the one users read.
- [.claude/PROJECT_CONTEXT.md](../.claude/PROJECT_CONTEXT.md) is the working session log. It holds architecture decisions with their reasoning, the current state, open next steps, and a longer version of the traps above.
- [additional-skills/README.md](../additional-skills/README.md) is the single sources and licensing document for the vendored skills.
- Every plugin carries its own `README.md` and `CHANGELOG.md`, and every test suite has a `README.md` covering its tiers in detail.

Plugin code depends on the MariaDB Shell MCP plugin, so it is licensed GPL-2.0, and each plugin ships an identical copy of the root `LICENSE`. Vendored skills keep whatever license their source repository set.
