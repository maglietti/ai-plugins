# Recipe: expose a schema over REST

You end up with a note-taking schema running on a throwaway MariaDB instance, a REST API in front of it, and a way to prove both exist. The whole thing is one prompt. The agent writes the SQL, spins up the server, runs the scripts, and verifies the result.

This workflow is covered by the `test_e2e_claude.py` end-to-end test, which drives a real Claude Code CLI through these exact steps and asserts the side effects. The last full run passed all six assertions in 5 minutes 10 seconds.

## What you need

- The `dev` plugin installed in Claude Code, since the `sql` variant omits the REST skills.
- A resolvable `mariadb-shell`, either on your `PATH` or pointed at by `MARIADB_SHELL_BIN`.
- One run of `mariadb-shell -- mcp setup`, so the MCP server is allowed to touch your working directory.

Work in an empty directory. The agent writes two SQL files and deploys a sandbox instance, and you want all of that somewhere you can delete afterwards.

## The prompt

Adapt the port and password, then paste this as a single message.

```text
Work in the current directory and complete every step in order.

1. Create a MariaDB database schema named notes-app for a note-taking app and
   store it in a notes-app.sql file.
2. Spin up a sandbox instance on port 3399 with root password 'test', connect to
   it and run notes-app.sql via the MCP server.
3. Create a second SQL script named notes-app-rest.sql that sets up the MariaDB
   REST Service for the notes-app schema: configure the REST metadata, create a
   REST service with the request path /notesApp, add a REST schema for the
   notes-app database schema, and add REST endpoints for its objects (a REST data
   mapping view for each table, and REST procedures/functions for any stored
   routines).
4. Run notes-app-rest.sql against the same sandbox via the MCP server.
5. Confirm the result by running SHOW REST SERVICES and the other SHOW REST
   commands (SHOW REST SCHEMAS, SHOW REST VIEWS, SHOW REST PROCEDURES, SHOW REST
   FUNCTIONS) against the sandbox via the MCP server.
```

Numbering the steps and saying "in order" is doing real work in that prompt. The REST DDL has a strict sequence, since a REST schema cannot exist before its service, and no endpoint can exist before its schema.

## Which skills carry each step

| Step | Skill | What it contributes |
| ---- | ----- | ------------------- |
| 1 | `mariadb-schema-create-script` | the session-settings Start Block, `CREATE SCHEMA IF NOT EXISTS` over `CREATE DATABASE`, a comment on every object |
| 1 | `mariadb-create-table` | MariaDB-specific column and table options |
| 3 | `mariadb-rest-service-create` | `CONFIGURE REST METADATA`, then `CREATE REST SERVICE`, `CREATE REST SCHEMA`, and the REST data mapping views |
| 5 | `mariadb-rest-service-show` | the `SHOW REST` family, which are `mariadb-shell` DDL extensions rather than server SQL |

Steps 2 and 4 are not skill work at all. They are MCP tool calls, using `sandbox.deploy` and then `db.connect` with `db.execute_sql`.

## Checking the result

Ask the agent to run `SHOW REST SERVICES`. A working run lists the `/notesApp` service, and `SHOW REST VIEWS` lists one endpoint per table in the schema.

To check without involving the agent, connect to the sandbox yourself and query the metadata schema:

```sql
SELECT url_context_root FROM mysql_rest_service_metadata.service;
SELECT COUNT(*) FROM mysql_rest_service_metadata.db_object;
```

Those tables keep their upstream MySQL REST Service names on purpose. The MariaDB REST Service is a fork, and the identifiers stayed put so existing tooling keeps working.

## Traps

**REST statements need one continuous session.** The `db.execute_sql_script` tool opens a fresh session for every statement, which breaks the REST grammar halfway through. Run the REST script through `db.execute_sql` on a single connection instead. Claude adapts to this on its own once it hits the failure, but you will save a few minutes by asking for `db.execute_sql` up front.

**A sandbox root password cannot be blank.** Pick something, as the recipe does with `test`, or `db.connect` refuses the connection afterwards.

**Clean up the sandbox.** It survives the conversation. Ask for `sandbox.stop` and then `sandbox.delete`, and reach for `sandbox.kill` if the stop fails, since `sandbox.delete` will not touch a running instance. An abandoned sandbox holds its port and its data directory under `~/mysql-sandboxes/<port>/`.

**The MCP allow-list gates file paths.** A path outside the allow-list turns into an elicitation prompt that a non-interactive run cannot answer, and the call fails. Run `mariadb-shell -- mcp setup` before you start.

## Next

- [Version a schema with MSM](schema-lifecycle.md) puts the same kind of schema under version control instead of a single script.
- `mariadb-rest-service-authorization` covers auth apps, users, roles, and the `GRANT REST` statements, which this recipe leaves out entirely.
- `mariadb-rest-service-update-endpoints` and `mariadb-rest-service-drop` handle changing and removing what you built here.
