# Getting started with `ai-plugins`

An orientation for working *on* this repository. If you only want to install the
plugins and use them, the root [README.md](../README.md) is the right document.
This one covers the parts you have to know before you change anything.

## 1. What the repository is

`ai-plugins` packages MariaDB agent **skills** (Markdown files named `SKILL.md`
that an AI coding agent loads on demand) together with the native
[`mariadb-shell`](https://github.com/mariadb-corporation/mariadb-shell) **MCP
server**, and ships that pair as an installable plugin for four coding agents:

| Harness | Directory | Variants shipped |
| ------- | --------- | ---------------- |
| Claude Code | `claude/` | `dev`, `sql`, `contributor` |
| Codex | `codex/` | `dev`, `sql`, `contributor` |
| OpenCode | `opencode/` | `dev`, `sql`, `contributor` |
| Pi (pi.dev) | `pi/` | `dev` only |

Ten plugin directories in total. The three variants differ only in what they
contain:

| Variant | Skills on disk today | MCP server | Skill sources |
| ------- | -------------------- | ---------- | ------------- |
| `dev` | 75 | yes | `mariadb-docs` agent-skills plus all of `additional-skills/` |
| `sql` | 47 | yes | the SQL-focused upstream layers plus `additional-skills/sql/` |
| `contributor` | 1 (`create-shell-plugin`) | no | the private `mariadb-shell` repo's `.claude/skills/` |

Skills are baseline **MariaDB 11.8 LTS**. The plugin package version and the
`mariadb-shell` minimum version are both `26.8.0` right now, and they are two
different numbers that happen to match (see section 7).

## 2. The mental model

Four facts explain most of the repository's shape.

**Skills are vendored, never authored in place.** Every
`<harness>/<variant>-plugin/skills/` tree is generated output. The real sources
are the upstream `mariadb-corporation/mariadb-docs` repository and this repo's
own `additional-skills/` folder. Editing a vendored copy works locally and gets
silently reverted by the next sync, so all edits go to the source plus a re-run
of `scripts/sync-skills.sh`.

**Every plugin has the same flat skill layout**, `skills/<skill-name>/SKILL.md`,
regardless of how upstream groups the skills. OpenCode discovers skills only one
directory deep, and pi's loader fails on any directory under `skills/` that has
no `SKILL.md`. Keep `skills/` holding nothing but skill directories,
`.skills-manifest.json`, and `skills-source.json`.

**The MCP server installs itself on first use.** The plugin ships
`scripts/mariadb-mcp-launcher.sh` (and a `.cmd` twin for native Windows), not a
binary. On launch it resolves a `mariadb-shell` in this order: `$MARIADB_SHELL_BIN`,
then a `mariadb-shell` on `PATH` at or above `MARIADB_SHELL_VERSION`, then an
install under `~/.local/bin`, and only if none of those satisfies the version
floor does it fetch the shell's own `install.sh` / `install.ps1` and run that. It
then execs the binary as an MCP server over stdio. Because stdout *is* the
JSON-RPC transport, every message the launcher or installer emits goes to stderr.

**The launcher is one file copied fourteen times.** Seven `.sh` copies and seven
`.cmd` copies, byte-identical. Edit `claude/dev-plugin/scripts/`, `cp` to the
others, and confirm with `shasum`. The `.cmd` copies must stay CRLF and ASCII;
`.gitattributes` pins that, because an LF batch file mis-parses the `goto` labels
and `^` line continuations the launcher relies on.

## 3. Prerequisites

For static work (skills, docs, structural tests) you need only Python 3.12 and
`pytest`. For everything else:

| Tool | Needed for |
| ---- | ---------- |
| `jq`, `curl` | `scripts/sync-skills.sh` |
| `gh` authenticated, or `GH_TOKEN` | syncing the `contributor` plugins and installing `mariadb-shell`, both from private repos |
| `mariadb-shell` on `PATH` | `run_tests.py`, the `db` tier, the `e2e` tiers |
| Docker | the `db` tier against a container instead of a sandbox instance |
| An authenticated harness CLI (`claude`, `codex`, `pi`) | the `e2e` tiers |
| `ANTHROPIC_API_KEY` or `OPENAI_API_KEY` | the `eval` tier |

The `mariadb-shell` repository is private today, so anonymous requests to it
answer 404 rather than 401. A missing token looks exactly like a missing
repository. If an install or sync fails with "not found", check your token first.

One machine-level setup step is easy to miss: the MCP server starts out allowed
to reach nothing. Run `mariadb-shell -- mcp setup` once per machine to give it an
allow-list.

## 4. Repository map

```text
ai-plugins/
├── .claude-plugin/marketplace.json    Claude Code marketplace entry
├── .agents/plugins/marketplace.json   Codex marketplace entry (Codex reads this one)
├── additional-skills/                 repo-local skill sources
│   ├── sql/                            1 skill, vendored into dev and sql
│   ├── rest/                           5 MariaDB REST Service skills, dev only
│   └── schema-management/              5 MSM lifecycle skills, dev only
├── claude/  codex/  opencode/  pi/    one directory per harness
│   ├── <variant>-plugin/              generated plugin: skills/, scripts/, MCP config
│   └── dev-plugin-test[s]/            that harness's pytest suite
├── scripts/
│   ├── sync-skills.sh                 vendors skills into every plugin
│   ├── set-plugin-version.sh          the plugin package version
│   └── set-mariadb-shell-version.sh   the mariadb-shell minimum version
├── run_tests.py                       runs every suite under the mariadb-shell Python
├── pytest-coverage.ini, .coveragerc   shared test and coverage config
├── package.json                       the pi manifest (its `pi` field points at pi/dev-plugin/)
└── .github/workflows/                 one workflow per harness
```

Two naming details will trip you up in shell globs. The Claude and pi suites are
`dev-plugin-tests` (plural); the Codex and OpenCode suites are `dev-plugin-test`
(singular). `run_tests.py` discovers them with the glob
`*/dev-plugin-test*/conftest.py`, which is why both spellings survive.

Pi is the odd harness. It has no marketplace file and no built-in MCP support. A
pi package is any directory carrying a `package.json` with a `pi` field, so the
**repo-root `package.json` is pi's manifest** and the whole repository installs
as one pi package. MCP reaches pi through the separately installed community
extension `pi-mcp-adapter`, which pi loads only as a package in its own right.

## 5. Install a plugin from your checkout

**Claude Code.** Add this directory as a marketplace, then install:

```text
/plugin marketplace add /Users/maglietti/Code/mariadb/ai-plugins
/plugin install dev@mariadb
```

**Codex.** Same two commands, then register the MCP server by hand. Codex 0.147
stores a plugin's `command` verbatim and expands no placeholder, so a plugin
cannot register a server that will actually start:

```sh
codex/dev-plugin/scripts/setup-codex-mcp.sh     # --remove to unregister
```

**OpenCode.** Merge the `mcp` block from `opencode/dev-plugin/opencode.json` into
your own `opencode.json`, point `MARIADB_DEV_PLUGIN` at the plugin directory, and
symlink the flat `skills/` into an OpenCode skills directory.

**Pi.** Install the adapter and this repository as two packages, then run the
setup command the extension registers:

```sh
pi install npm:pi-mcp-adapter
pi install .                    # from the repo root
```

```text
/mariadb-mcp-setup              # writes ~/.config/mcp/mcp.json
/mariadb-mcp-setup --project    # or ./.mcp.json for one project
```

## 6. Changing a skill

```sh
scripts/sync-skills.sh          # latest upstream default branch
scripts/sync-skills.sh <ref>    # a specific tag, branch, or commit
```

The script downloads the upstream `agent-skills/` tree once, copies the selected
skills into each plugin's `skills/` directory, rewrites `.skills-manifest.json` to
flat paths while preserving each skill's layer grouping, and records the resolved
commit SHA in each plugin's `skills-source.json`. Which subfolders land where is
controlled by two variables near the top of the script: `ADDITIONAL_SUBDIRS`
lists this repo's subfolders, and `SQL_INCLUDE_LAYERS` names the layers the `sql`
variant takes.

The `contributor` plugins come from a second source, the private `mariadb-shell`
repository. Without `GH_TOKEN` or a working `gh auth token`, that step prints a
warning and skips, and the `dev` and `sql` sync still succeeds.

Two habits worth forming. First, editing `additional-skills/` without re-running
the sync ships stale text to every plugin, and nothing else notices: the manifest
still matches disk and the stale copy still parses. That went unnoticed for four
days once. The static test `test_additional_skill_matches_its_source` now catches
it. Second, a sync at the upstream default branch head can pull in unrelated
churn. Read `git diff --shortstat` and one sample diff before concluding that a
432-file sync means 432 content revisions.

## 7. Changing a version

The two versions move independently and each has its own script. Do not edit the
files by hand; both scripts reach into more places than you will remember.

```sh
scripts/set-plugin-version.sh 26.9.0        # the plugin package version
scripts/set-mariadb-shell-version.sh 26.9.0 # the mariadb-shell minimum
```

`set-plugin-version.sh` rewrites the `version` field in each plugin manifest and
the `Version **x.y.z**` line in each plugin README, across all four harnesses,
plus the repo-root `package.json` (pi has no per-plugin manifest).
`set-mariadb-shell-version.sh` rewrites `MARIADB_SHELL_VERSION` across the 35
files that carry it: the launchers, `.mcp.json`, `opencode.json`,
`setup-pi-mcp.sh`, `setup-codex-mcp.{sh,cmd}`, and the READMEs' prose. That prose
line has to stay on one line or the substitution stops matching it.

This script's reach has lagged behind new files three times (when pi arrived,
when the disabled launchers existed, and when the setup scripts gained a default).
Whenever a file gains a version site, check that the script lists it.

Note that `MARIADB_SHELL_VERSION` is a **floor for accepting a binary already on
disk**, never a trigger to look for a newer one. A machine with 26.8.0 installed
will keep using it after 26.9.0 ships. A download happens only when nothing on
the machine meets the floor.

## 8. Running the tests

Four tiers, the same markers in all four suites:

| Tier | Marker | Needs | Checks |
| ---- | ------ | ----- | ------ |
| 1 | `static` | nothing | frontmatter, manifest against disk, cross-references, SQL fences, the statement-skill contract |
| 2 | `db` | `mariadb-shell` plus a server binary | the skills' recommended DDL runs on a live server with the documented effect |
| 3 | `eval` | an LLM API key | the skill steers the model toward the MariaDB-preferred form |
| 4 | `e2e` | the harness CLI, authenticated | drives the real CLI with the plugin loaded, then asserts side effects |

Tiers 3 and 4 are opt-in and deselected by default. Tiers 1 and 2 are identical
everywhere. The opt-in tiers differ by harness: Claude and Codex have both an
`eval` tier and two `e2e` workflows (the REST workflow and the MSM schema
lifecycle); OpenCode has `eval` only; pi has `e2e` only, because pi has no SDK of
its own to prompt, which makes driving the `pi` binary the behavioral test.

The preferred runner is the repo-root `run_tests.py`. It executes every suite
with the Python that ships inside `mariadb-shell` (`mariadb-shell --pym pytest`),
so the tests run against the same interpreter and packages as the MCP server they
exercise, and it appends coverage across the runs into one report.

```sh
./run_tests.py                  # every suite, default tiers (static and db)
./run_tests.py claude           # one suite, named after its top-level directory
./run_tests.py -m static        # one tier, all suites
./run_tests.py claude -m e2e    # the opt-in end-to-end tier
./run_tests.py -k manifest      # only tests matching a pattern
./run_tests.py -- --lf -x       # anything after `--` goes to pytest

npm test                        # the same, plus test:static, test:db, test:eval, test:e2e
```

The binary comes from `--shell`, then `$MARIADB_SHELL`, then `PATH`, and its
directory is prepended to the `PATH` the suites see so the `e2e` tier's launcher
resolves the same one. `--no-install` skips installing each suite's
`requirements.txt`, `--no-coverage` skips measurement, and `--userhome` relocates
the shell's user config home (by default the real one is left alone).

To drive pytest directly instead, work inside a suite directory:

```sh
cd claude/dev-plugin-tests
pip install -r requirements.txt
pytest -m static
```

The `db` tier needs no server running beforehand. It deploys a throwaway sandbox
instance on a free port through the MCP server's `sandbox.*` tools and deletes it
afterwards, with the shell's config home isolated to a temp directory for the run.
Setting any of `MARIADB_HOST`, `MARIADB_PORT`, `MARIADB_USER`, or
`MARIADB_PASSWORD` switches the tier onto an already-running server instead, which
is what CI and each suite's `docker-compose.yml` do.

As of the last full local run: `static` is 602 tests in the Claude, Codex, and
OpenCode suites and 611 in pi (the 9 extra guard pi-specific wiring); the default
tiers together are 618; `e2e` is 12 for Claude. Reports land in `test-results/`
and `htmlcov/`, both git-ignored.

## 9. What CI runs

Four workflows, one per harness, in `.github/workflows/`. Each runs `static` on
every push and pull request, `db` on pull requests against a `mariadb:11.8`
service container, and (for the three suites that have the tier) `eval` nightly
or on demand. The `e2e` tiers are local only, since they need an authenticated
CLI and a resolvable `mariadb-shell`, neither of which a GitHub runner has.

CI still calls `pytest` per suite with `actions/setup-python` rather than
`run_tests.py`, deliberately: runners have no `mariadb-shell`, so the
shell-Python runner is a local path. The per-suite `pyproject.toml` files stay
authoritative for the CI runs, and `pytest-coverage.ini` for the local ones.
Those two sets of markers have to stay in sync by hand.

## 10. Traps that have already cost time

- **Batch files must be CRLF, and a license header must go after `@echo off`.**
  Command echo is on until `@echo off` runs, so `rem` lines above it print to
  stdout, which for a launcher is the MCP transport.
- **Codex spawns MCP servers with a filtered environment.** Exported variables
  such as `MARIADB_SHELL_USER_CONFIG_HOME` do not reach the server, so it falls
  back to the real `~/.mariadb-shell` and its allow-list.
- **`subprocess.run` on `codex exec` must pass `stdin=subprocess.DEVNULL`.** Codex
  reads a piped stdin as extra prompt input and blocks until the timeout, with an
  empty event stream as the only symptom.
- **Pi ignores project-local packages unless the run passes `--approve`.**
  `pi install -l` reports success either way, and without approval the skills are
  simply absent with no error.
- **Seed the MCP allow-list with both macOS path spellings.** `$TMPDIR` yields
  `/var/folders/...` while `Path.resolve()` yields `/private/var/folders/...`, and
  the path guard compares strings.
- **Do not assert model obedience and call it wiring.** Two tests have gone red
  demanding a fixed output shape while the plugin underneath was fine. Assert on
  the side effects the plugin produces, not on the words the model chooses.
- **Test a resolution chain by measuring the branch, not the outcome.** A probe
  keyed on "did the shell launch" reported success for every version, because a
  rejected binary is still launched by the install-failure fallback. Assert on the
  launcher's own stderr line instead.
- **Piping a test run through `tee` reports tee's exit code, not pytest's.** Read
  the pytest summary line.

## 11. Where the deeper context lives

- [README.md](../README.md) is the install-facing document and the one users read.
- [.claude/PROJECT_CONTEXT.md](../.claude/PROJECT_CONTEXT.md) is the working
  session log: architecture decisions with their reasoning, current state, open
  next steps, and a longer version of section 10.
- [additional-skills/README.md](../additional-skills/README.md) is the single
  sources and licensing document for the vendored skills.
- Each plugin has its own `README.md` and `CHANGELOG.md`, and each test suite has
  a `README.md` covering its tiers in detail.

Plugin code depends on the MariaDB Shell MCP plugin and is licensed **GPL-2.0**;
each plugin ships an identical copy of the root `LICENSE`. Vendored skills keep
the license of the repository they came from.
