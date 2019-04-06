
Overview[¶](#overview "Permanent link")
---------------------------------------

Pushing database code to production is not as easy as replacing some files. We need to generate the SQL DDL (Data Definition Language) statements that take our database code from the old version to the new one. This is done through "migration" files. An excellent tool to manage and track those files is [Sqitch](http://sqitch.org/).

This would be enough if we would have used the database as a "dumb store" since, after the first version is deployed, there would be very rare and few changes to the schema, so writing those files by hand would not be that hard. We however, have a lot of the code in the database that guarantees the integrity of our data. We've also split that code into multiple files and directories. Having to remember what changed and where would be impossible so this is were the [subzero-cli](https://github.com/subzerocloud/subzero-cli) comes in handy again. You do not have to remember what files you changed in order to implement a feature, you change any sql file you need, and at the end, when you have your feature ready to be pushed to production, you just run one command, and that migration file will be generated for you.

But anyway, let's see how that process actually goes

Initial Database State[¶](#initial-database-state "Permanent link")
-------------------------------------------------------------------

Right now our code is split into multiple files. We need to create the first migration that has the initial state of our database.

Make sure your stack is up (we need it running when creating migrations), if not, bring it up with `docker-compose up -d`

Create the first migration

subzero migrations init

If all went well, you should see this output

Created sqitch.conf
Created sqitch.plan
Created deploy/
Created revert/
Created verify/
Created deploy/0000000001-initial.sql
Created revert/0000000001-initial.sql
Created verify/0000000001-initial.sql
Added "0000000001-initial" to sqitch.plan

Note

You might be tempted to start creating migrations from the early stages of iterating on your API in order to be able to push changes to a remote deployment used for testing. You should avoid that. During the early stages you will have a lot of changes in your migration files and it will be a lot of work checking their correctness. Until you actually go to production, you should do full dumps of your dev database each time and use that for your remote test system.

Adding data to migrations[¶](#adding-data-to-migrations "Permanent link")
-------------------------------------------------------------------------

If you look at the `db/migrations/deploy/0000000001-initial.sql` file, you'll see only statements that will create all your entities, you will not see any statement that will add the actual data in the tables that you currently have in your dev database. Usually this is exactly what you want in regard to the data, however there are still some places where the data in the tables is more like a configuration as opposed to user generated data so it also needs to be reflected in a migration. Since it's impossible to know which is which, this will be a manual step.

Create an empty migration

subzero migrations add --no-diff --note "initial dataset" data

# result
Created deploy/0000000002-data.sql
Created revert/0000000002-data.sql
Created verify/0000000002-data.sql
Added "0000000002-data" to sqitch.plan

Change `db/migrations/deploy/0000000002-data.sql` to this

\-- Deploy app:0000000002-data to pg

BEGIN;

SET search\_path \= settings, pg\_catalog, public;

COPY secrets (key, value) FROM stdin;
jwt\_lifetime    3600
auth.default\-role   webuser
auth.data\-schema    data
auth.api\-schema api
\\.

INSERT INTO secrets (key, value) VALUES ('jwt\_secret', gen\_random\_uuid());

COMMIT;

And now the revert migration `db/migrations/revert/0000000002-data.sql` to this

\-- Revert app:0000000002-data from pg

BEGIN;

SET search\_path \= settings, pg\_catalog;
TRUNCATE secrets;

COMMIT;

Incremental changes/migrations[¶](#incremental-changesmigrations "Permanent link")
==================================================================================

After you've finished work on some feature, you need to create the migration file. Let's check that process.

Lets make a change to `todos` view. Change [this line](https://github.com/subzerocloud/subzero-starter-kit/blob/master/db/src/api/todos.sql#L9)

#before
select id, todo, private, (owner\_id \= request.user\_id()) as mine from data.todo;

#after
select ('#' || id::text) as id, ('do this: ' || todo) as todo, private, (owner\_id \= request.user\_id()) as mine from data.todo;

Save the file (the `subzero dashboard` needs to be always running when making changes)

Now let's create the migration

subzero migrations add --note "change todos view" todos

\# result
Starting temporary Postgre database
325638e3b549cc6eb41a87097295d6203629459782a2914ccb4fdf4ce3a0594e
Waiting for it to load
PostgreSQL init process complete; ready for start up.

Created deploy/0000000003-todos.sql
Created revert/0000000003-todos.sql
Created verify/0000000003-todos.sql
Added "0000000003-todos" to sqitch.plan


Diffing sql files
ATTENTION: Make sure you check deploy/0000000003-todos.sql for correctness, statement order is not handled!

Now open `db/migrations/deploy/0000000003-todos.sql` file and see the result

START TRANSACTION;

SET search\_path \= api, pg\_catalog;

DROP VIEW todos;

CREATE VIEW todos AS
    SELECT ('#'::text || (todo.id)::text) AS id,
    ('do this: '::text || todo.todo) AS todo,
    todo.private,
    (todo.owner\_id \= request.user\_id()) AS mine
   FROM data.todo;
REVOKE ALL ON TABLE todos FROM webuser;
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLE todos TO webuser;
REVOKE ALL(id) ON TABLE todos FROM anonymous;
GRANT SELECT(id) ON TABLE todos TO anonymous;
REVOKE ALL(todo) ON TABLE todos FROM anonymous;
GRANT SELECT(todo) ON TABLE todos TO anonymous;

COMMIT TRANSACTION;

`subzero-cli` detected that only one view changed since your last migration and created the appropriate DDL statements.

Note

In the example above you've probably noticed the `ATTENTION:` bit and wondering what's that about. Internally, the cli uses [apgdiff](https://www.apgdiff.com) to auto generate migrations files, and while it does a good job most of the time, it's not 100% correct 100% of the time since generating DDL while making sure to preserve the current state is a complex task. For this reason, we recommend that when working on big features that involve lots of changes to different database entities, you generate a few smaller migrations instead of a big one at the end. This will make it easier to verify the correctness of the generated DDL statements. Once you are ready to push your big feature to production you can merge those migration files to a single one.

