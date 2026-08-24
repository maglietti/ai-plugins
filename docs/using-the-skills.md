# Using the MariaDB skills

You installed the `dev` plugin and your agent looks exactly the same as before. This guide explains what actually changed, how to get value out of it, and how to tell whether any of it is working. It assumes Claude Code. The other three harnesses load the same skills, and the ideas carry over even where the commands do not.

## You never call a skill

This is the one thing to understand before anything else. A skill is not a command you invoke. It is a Markdown file whose YAML frontmatter carries a `description` that tells the agent when to reach for it. Here is the real trigger from `mariadb-create-table`, trimmed to its last clause:

```yaml
description: "MariaDB-specific syntax and behavior for CREATE TABLE ...
  Use when writing, generating, or reviewing CREATE TABLE statements that target MariaDB."
```

You write "add an invisible `created_at` column to the orders table" and the agent matches that intent against every installed skill's `description`, then loads the ones that fit. Nothing in your prompt names a skill, and nothing needs to. So the practical question is never "which skill do I call", it is "did I describe the work in terms the trigger recognizes".

Naming a skill outright does nudge the agent toward it, which is useful when you know exactly what you want. Treat it as a strong hint rather than a guarantee.

## What your agent gained

The `dev` plugin installs 75 skills. The `sql` plugin installs the 47 of them that concern SQL. Knowing roughly what is in the catalog is what lets you recognize a task as one the agent is now good at.

| Layer | Count | Covers |
| ----- | ----- | ------ |
| Statements | 31 | `mariadb-create-table`, `mariadb-alter-table`, `mariadb-select`, `mariadb-insert`, and the rest of the DDL and DML surface |
| Connectors | 14 | seven connector families (Python, Java, Node.js, ODBC, R2DBC, C++, C), each split into an install skill and a usage skill |
| Functions | 10 | `mariadb-json-functions`, `mariadb-string-functions`, `mariadb-date-time-functions`, `mariadb-numeric-functions`, and siblings |
| Topical | 5 | `mariadb-features`, `mariadb-query-optimization`, `mariadb-system-versioned-tables`, `mariadb-vector`, `mysql-to-mariadb` |
| Client tools | 4 | `mariadb-dump`, `mariadb-import`, `mariadb-client`, `mariadb-binlog` |
| Repo-local | 11 | the MariaDB REST Service (5), the MariaDB Schema Management lifecycle (5), and `mariadb-schema-create-script` |

Every skill is written against MariaDB 11.8 LTS. The value concentrates where MariaDB diverges from what a model already half-knows about MySQL. Ask for a table and you get `CREATE OR REPLACE` instead of a `DROP` and a race. Ask about row history and you get system-versioned tables rather than a hand-rolled audit trigger. Ask about vectors and you get the built-in `VECTOR` type rather than an extension that does not exist here.

## Phrasing work so the right skill fires

The triggers key on the operation and the target, so name both.

| Instead of | Write |
| ---------- | ----- |
| "make a users table" | "write the `CREATE TABLE` for a MariaDB users table with an invisible audit column" |
| "how do I connect from Python" | "set up the MariaDB Connector/Python client and show a parameterized insert" |
| "this query is slow" | "run `EXPLAIN` on this MariaDB query and propose an index" |
| "track changes to this table" | "use MariaDB system-versioned tables to keep row history for `orders`" |
| "give me the schema" | "generate a MariaDB schema create script for a note-taking app" |

The pattern is the same in each row. Say the MariaDB artifact you want, not the outcome you have in mind. The last row matters more than it looks, because "schema create script" is what pulls in `mariadb-schema-create-script` and its mandatory session-settings block.

## Checking that a skill actually fired

Be skeptical here, because the honest answer is that most harnesses give you no direct proof. The pi harness exposes no skill data in its event stream at all. What you get instead is behavioral evidence, and one probe is worth more than any amount of asking the agent what it knows.

Ask for a schema create script. If the skills loaded, the output opens with the session-settings Start Block that `mariadb-schema-create-script` mandates:

```sql
-- Save current session settings and disable checks for faster, safer bulk import
SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS=0;
SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0;
SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='NO_AUTO_VALUE_ON_ZERO,STRICT_TRANS_TABLES';
```

No model produces that from general SQL knowledge. Its presence is the signal, and the end-to-end test suite uses exactly this check as its own proof that the plugin reached the model. A script that starts straight into `CREATE SCHEMA` means the skills are not loading, so go back to the install steps in the root [README.md](../README.md).

Two weaker checks are still worth knowing. Run `/plugin` in Claude Code to confirm the plugin is installed at all, and ask the agent which MariaDB skills it has available. The second one reports what the agent believes rather than what is loaded, so trust the Start Block probe over the answer.

## The part you do drive directly

The MCP server is the other half of the plugin and it behaves nothing like the skills. It gives the agent live access to a database through tools it calls on your behalf. There are three families:

- `sandbox.deploy`, `sandbox.start`, `sandbox.stop`, `sandbox.kill`, and `sandbox.delete` manage throwaway MariaDB instances, so you can ask for a real server without installing one.
- `db.connect`, `db.execute_sql`, `db.execute_sql_script`, and `db.close` run SQL against a connection.
- The `msm.*` family drives versioned schema projects, which the [schema lifecycle recipe](recipes/schema-lifecycle.md) covers.

Run `mariadb-shell -- mcp setup` once per machine before any of this works. The server starts out allowed to reach nothing, and every file-touching tool checks a path allow-list that begins empty.

Three behaviors will confuse you the first time. The `db.connect` tool wants a bare connection string like `root@127.0.0.1:3306`, and rejects a `mariadb://` URL as an unconfigured connection. The `db.execute_sql_script` tool opens a fresh session per statement, so anything needing one continuous session has to go through `db.execute_sql` instead. A sandbox root password cannot be blank, and `sandbox.delete` refuses to touch a running instance, which makes `sandbox.kill` your fallback when a stop fails.

## When nothing fires

Work down this list.

1. Confirm the plugin is installed, with `/plugin` in Claude Code.
2. Run the Start Block probe above. It separates "skills are absent" from "the agent chose not to use one".
3. Re-read your prompt for the MariaDB artifact name. "Fix my database" matches no trigger; "rewrite this `ALTER TABLE` for MariaDB" matches one directly.
4. Check that you installed the variant you meant. The `sql` plugin has no connector or client-tool skills in it, so a Connector/Python question finds nothing to load.
5. For MCP failures specifically, check `mariadb-shell -- mcp setup` and the path allow-list before suspecting the plugin.

## Recipes

Two multi-step workflows are documented end to end, and both are backed by passing end-to-end tests that drive a real Claude Code CLI:

- [Expose a schema over REST](recipes/rest-service.md) builds a schema, runs it against a live sandbox, and puts a REST API in front of it.
- [Version a schema with MSM](recipes/schema-lifecycle.md) takes a schema through two releases and the migration between them.
