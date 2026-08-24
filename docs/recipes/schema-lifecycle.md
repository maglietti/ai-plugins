# Recipe: version a schema with MSM

MariaDB Schema Management (MSM) treats a schema the way you already treat code, with versions, releases, and a migration between them. This recipe takes a note-taking schema from nothing to version 1.0.0, adds features for version 1.1.0, and produces a deployment script that can either create the schema fresh or upgrade an existing install to match.

The `test_e2e_msm_claude.py` end-to-end test drives a real Claude Code CLI through these steps and checks the files that come out. The last full run passed all six assertions in 11 minutes 23 seconds.

Nothing here touches a database. The `msm.prepare_release` and `msm.generate_deployment_script` tools are on-disk operations, so you can run the whole recipe without a server anywhere in sight, then deploy the output whenever you like.

## What you need

- The `dev` plugin installed in Claude Code, since the `sql` variant omits the schema-management skills.
- A resolvable `mariadb-shell`, either on your `PATH` or pointed at by `MARIADB_SHELL_BIN`.
- One run of `mariadb-shell -- mcp setup`, because the `msm.*` tools write files and check the path allow-list before they do.

## The prompt

```text
Use the MariaDB Schema Management (MSM) tools of the mariadb-shell MCP server to
manage a note-taking app as a versioned schema project. Work entirely inside the
current directory and complete every step in order.

1. Create an MSM schema project for a schema named notes_app in the current
   directory. For the initial version 1.0.0, author only two tables, `user` and
   `note`, plus a VIEW named `user_activity` that lists users ordered by their
   activity (their number of notes). Put the tables in the non-idempotent create
   section and the `user_activity` VIEW in the idempotent create section.
2. Prepare the 1.0.0 release and generate its deployment script.
3. Develop the next version on top of 1.0.0: add support for notebooks and tags
   with new `notebook` and `tag` tables, and add a VIEW named `notes_details`
   that joins notes with their notebook and tags.
4. Prepare version 1.1.0. Fill the previous->1.1.0 update script with the
   migration, the new `notebook` and `tag` tables in the non-idempotent update
   section and the `notes_details` VIEW in the idempotent update section, then
   generate the 1.1.0 deployment script.
```

The phrases "non-idempotent create section" and "idempotent create section" are the important ones. They map onto numbered sections in the MSM file format, and getting an object into the wrong section is the most common way to break a deployment.

## The section model

An MSM script is divided into numbered sections, and the number decides both where a statement runs and whether it can run twice.

| Section | Holds | Why the split matters |
| ------- | ----- | --------------------- |
| 140 | tables and other objects you create once | becomes a stored-procedure body in the deployment script, so it takes plain `;` terminators and no `DELIMITER` |
| 150 | views and routines, re-created every time | idempotent by construction, using `CREATE OR REPLACE` |
| 240 | `ALTER TABLE` statements and every drop, for an upgrade | the migration half of section 140 |
| 250 | the same views as 150, re-created after the migration | re-runs cleanly after an upgrade |

Tables go in 140 and views go in 150. A `CREATE TABLE` that lands in 150 will fail the second time the deployment script runs, which is exactly when you least want to find out.

## What gets written

```text
notes_app.msm.project/
├── msm.project.json
├── development/
│   └── notes_app_next.sql                          the working script
└── releases/
    ├── versions/    notes_app_1.0.0.sql            full snapshot per release
    ├── updates/     notes_app_1.0.0_to_1.1.0.sql   the migration, filled by hand
    └── deployment/  notes_app_deployment_1.1.0.sql generated, never edited
```

## The rule everyone gets wrong

The `msm.prepare_release` tool creates an **empty** update script. You have to fill its migration sections before you generate the deployment script, because the deployment script embeds them to upgrade existing installs. Get that order backwards and you produce a deployment script that installs cleanly on a fresh database and silently fails to upgrade anything.

The order that works is prepare the release, fill sections 240 and 250, then generate the deployment script. Step 4 of the prompt spells this out for exactly that reason.

## Checking the result

Look at the files rather than asking the agent whether it worked.

```sh
ls notes_app.msm.project/releases/versions      # notes_app_1.0.0.sql, notes_app_1.1.0.sql
ls notes_app.msm.project/releases/deployment    # deployment scripts for both versions
grep -c 'CREATE TABLE' notes_app.msm.project/releases/updates/notes_app_1.0.0_to_1.1.0.sql
```

That last count should be 2, one each for `notebook` and `tag`. A count of zero means the update script was generated but never filled, which is the failure mode described above and the one the end-to-end test checks for by stripping comments before it looks.

## Traps

**The update script looks finished when it is empty.** It ships full of explanatory comments, so a glance suggests content. Strip the comments before you judge it, the way the test does.

**Section numbers are not interchangeable.** Sections 140, 240, 170, and 270 become stored-procedure bodies, so they take plain `;` terminators and no `DELIMITER` statement. Sections 130, 150, 230, 250, 190, and 290 are top level and use `DELIMITER %%`.

**Deployment scripts are generated output.** Never hand-edit one. Fix the section it came from and regenerate.

**The `msm.deploy_schema` tool needs an open connection.** Every other tool in the family works on files alone, so this is the one step that requires `db.connect` first.

## Next

- [Expose a schema over REST](rest-service.md) puts a REST API in front of a schema like this one.
- The `mariadb-schema-management` skill is the overview, and `mariadb-schema-management-develop` covers breaking a large development script into `SOURCE` files.
- The `mariadb-alter-table` skill supplies the DDL that section 240 needs.
