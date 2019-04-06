
API Core
========

Schemas[¶](#schemas "Permanent link")
-------------------------------------

### Data Schema[¶](#data-schema "Permanent link")

We'll start by creating all the tables that will hold the data and place them in a separate schema called `data`. There is no particular requirement to have the tables in the `data` schema nor do they all have to be in a single schema, this is just a convention which we think works out well, it will give us an understanding of how your models are decoupled from the API you expose.

Usually, you would create one file per table, but to make it easier to copy/paste the code, we'll add it all in a single file `db/src/data/tables.sql`

create table client (
  id           serial primary key,
  name         text not null,
  address      text,
  user\_id      int not null references "user"(id),
  created\_on   timestamptz not null default now(),
  updated\_on   timestamptz
);
create index client\_user\_id\_index on client(user\_id);

create table project (
  id           serial primary key,
  name         text not null,
  client\_id    int not null references client(id),
  user\_id      int not null references "user"(id),
  created\_on   timestamptz not null default now(),
  updated\_on   timestamptz
);
create index project\_user\_id\_index on project(user\_id);
create index project\_client\_id\_index on project(client\_id);

create table task (
  id           serial primary key,
  name         text not null,
  completed    bool not null default false,
  project\_id   int not null references project(id),
  user\_id      int not null references "user"(id),
  created\_on   timestamptz not null default now(),
  updated\_on   timestamptz
);
create index task\_user\_id\_index on task(user\_id);
create index task\_project\_id\_index on task(project\_id);

create table project\_comment (
  id           serial primary key,
  body         text not null,
  project\_id   int not null references project(id),
  user\_id      int not null references "user"(id),
  created\_on   timestamptz not null default now(),
  updated\_on   timestamptz
);
create index project\_comment\_user\_id\_index on project\_comment(user\_id);
create index project\_comment\_project\_id\_index on project\_comment(project\_id);

create table task\_comment (
  id           serial primary key,
  body         text not null,
  task\_id      int not null references task(id),
  user\_id      int not null references "user"(id),
  created\_on   timestamptz not null default now(),
  updated\_on   timestamptz
);
create index task\_comment\_user\_id\_index on task\_comment(user\_id);
create index task\_comment\_task\_id\_index on task\_comment(task\_id);

We added `user_id` column on each table because it will help us later with enforcing access rights for each row with a simple rule (instead of doing complicated joins). Having two separate tables for comments (task\_comment, project\_comment) is not the best schema design and it's a bit forced but it will be useful for this tutorial to showcase how the API can be decoupled from the underlying tables.

Change the last lines in `db/src/data/schema.sql` to look like this (the `...` are meant to signify there is something above)

\-- ...
\-- import our application models
\\ir todo.sql
\\ir tables.sql

_Note: the todo.sql definition came with the starter kit, we'll remove it later_

### Api Schema[¶](#api-schema "Permanent link")

This schema (`api`) will be the schema that we will be exposing for REST and GraphQL. It will contain only views and stored procedures.

Once again, we'll place all definitions in a single file called `db/src/api/views_and_procedures.sql`

create or replace view clients as
select id, name, address, created\_on, updated\_on from data.client;

create or replace view projects as
select id, name, client\_id, created\_on, updated\_on from data.project;

create or replace view tasks as
select id, name, completed, project\_id, created\_on, updated\_on from data.task;

create or replace view comments as
select 
  id, body, 'project'::text as parent\_type, project\_id as parent\_id, 
  project\_id, null as task\_id, created\_on, updated\_on
from data.project\_comment
union
select id, body, 'task'::text as parent\_type, task\_id as parent\_id,
  null as project\_id, task\_id, created\_on, updated\_on
from data.task\_comment;

Edit the last lines of `db/src/api/schema.sql` to look like this.

\-- ...
\-- our endpoints
\\ir todos.sql
\\ir views\_and\_procedures.sql

Notice how we were able to expose only the fields we wanted and only the tables we wanted. We could have renamed the columns also, for example maybe your front end devs turn their faces in disgust when they see something like `first_name`, no worries, you can make them happy and rename that column in the view to `firstName` and everything will work just fine. We also inherited a `data` schema which has two separate tables to hold the comments but for the API, we were able to expose them using a single entity and hide the underlying tables.

This is how you can decouple your implementation details (the underlying source tables) from the API you expose, using views or in more complex cases, stored procedures.

### Sample Data[¶](#sample-data "Permanent link")

It's no fun testing with empty tables, let's add some sample data.

_If you have a lot of models/tables it can get tediouse to generate a meaningfull dataset to test your api with, especially when you want to test the performance with a couple of millions of rows in the tables. You can try [datafiller](https://www.cri.ensmp.fr/people/coelho/datafiller.html) utility to generate some data._

Add these statements at the end of `db/src/sample_data/data.sql`

\-- ...
set search\_path \= data, public;
\\echo # filling table client (3)
COPY client (id,name,address,user\_id,created\_on,updated\_on) FROM STDIN (FREEZE ON);
1   Apple   address\_1\_  1   2017\-07\-18 11:31:12 \\N
2   Microsoft   address\_1\_  1   2017\-07\-18 11:31:12 \\N
3   Amazon  address\_1\_  2   2017\-07\-18 11:31:12 \\N
\\.

\\echo # filling table project (4)
COPY project (id,name,client\_id,user\_id,created\_on,updated\_on) FROM STDIN (FREEZE ON);
1   MacOS   1   1   2017\-07\-18 11:31:12 \\N
2   Windows 2   1   2017\-07\-18 11:31:12 \\N
3   IOS 1   1   2017\-07\-18 11:31:12 \\N
4   Office  2   1   2017\-07\-18 11:31:12 \\N
\\.

\\echo # filling table task (5)
COPY task (id,name,completed,project\_id,user\_id,created\_on,updated\_on) FROM STDIN (FREEZE ON);
1   Design a nice UI    TRUE    1   1   2017\-07\-18 11:31:12 \\N
2   Write some OS code  FALSE   1   1   2017\-07\-18 11:31:12 \\N
3   Start aggressive marketing  TRUE    2   1   2017\-07\-18 11:31:12 \\N
4   Get everybody to love it    TRUE    3   1   2017\-07\-18 11:31:12 \\N
5   Move everything to cloud    TRUE    4   1   2017\-07\-18 11:31:12 \\N
\\.

\\echo # filling table project\_comment (2)
COPY project\_comment (id,body,project\_id,user\_id,created\_on,updated\_on) FROM STDIN (FREEZE ON);
1   This is going to be awesome 1   1   2017\-07\-18 11:31:12 \\N
2   We still have the marketshare, we should keep it that way   2   1   2017\-07\-18 11:31:12 \\N
\\.

\\echo # filling table task\_comment (2)
COPY task\_comment (id,body,task\_id,user\_id,created\_on,updated\_on) FROM STDIN (FREEZE ON);
1   Arn't we awesome?   1   1   2017\-07\-18 11:31:12 \\N
2   People are going to love the free automated install when they see it in the morning 3   1   2017\-07\-18 11:31:12 \\N
\\.

\-- 
ALTER SEQUENCE client\_id\_seq RESTART WITH 4;
ALTER SEQUENCE project\_id\_seq RESTART WITH 5;
ALTER SEQUENCE task\_id\_seq RESTART WITH 6;
ALTER SEQUENCE project\_comment\_id\_seq RESTART WITH 3;
ALTER SEQUENCE task\_comment\_id\_seq RESTART WITH 3;

\-- 
ANALYZE client;
ANALYZE project;
ANALYZE task;
ANALYZE project\_comment;
ANALYZE task\_comment;

While we are at it, let's also change the `db/src/sample_data/reset.sql` file that is used when running tests

BEGIN;
\\set QUIET on
\\set ON\_ERROR\_STOP on
set client\_min\_messages to warning;
set search\_path \= data, public;
truncate todo restart identity cascade;
truncate user restart identity cascade;
truncate client restart identity cascade;
truncate project restart identity cascade;
truncate task restart identity cascade;
truncate project\_comment restart identity cascade;
truncate task\_comment restart identity cascade;
\\ir data.sql
COMMIT;

Since we have some data in the tables let's see how far did we get.

curl -H "Authorization: Bearer $JWT\_TOKEN" http://localhost:8080/rest/clients?select\=id,name

{"hint":null,"details":null,"code":"42501","message":"permission denied for relation clients"}

Believe it or not, this is a good thing :), we added a bunch of tables and views but this does not mean the API users can just access them, we need to explicitly state who can access what resource.

Securing your API[¶](#securing-your-api "Permanent link")
---------------------------------------------------------

### Access rights to API entities[¶](#access-rights-to-api-entities "Permanent link")

Let's start by giving the authenticated API users the ability to access clients/projects/tasks endpoints. This means we need to grant them rights to the views in the api schema.

Add the following statements at the end of `db/src/authorization/privileges.sql` file

\-- ...
grant select, insert, update, delete 
on api.clients, api.projects, api.tasks, api.comments
to webuser;

Let's try that request again

curl -H "Authorization: Bearer $JWT\_TOKEN" http://localhost:8080/rest/clients?select\=id,name

\[{"id":1,"name":"Apple"},{"id":2,"name":"Microsoft"},{"id":3,"name":"Amazon"}\]

Look at that :)!

So we have our API, nice and decoupled, but as things stand now, anyone (that can login) is able to modify the existing data and see all of it (Amazon is not the client of our currently logged in user `alice`). I would not want to use a system where everyone can see all my clients and projects. We will fix that but before we get to it, an interlude.

### The magic "webusers" role and the mystery $JWT token[¶](#the-magic-webusers-role-and-the-mystery-jwt-token "Permanent link")

You are probably wondering "What's up with that, where does that role come from and why this hard coded `$JWT` token just works on my system? It does not seem very secure to me.".

Well, there is nothing magic about it, the `webuser` role is created in [roles.sql](https://github.com/subzerocloud/subzero-starter-kit/blob/master/db/src/authorization/roles.sql#L35) based on the definition or the [user\_role](https://github.com/subzerocloud/subzero-starter-kit/blob/master/db/src/data/schema.sql#L7) type. It can be a simple `create role ...` statement but we are using the functionality of the [auth lib](https://github.com/subzerocloud/subzero-starter-kit/tree/master/db/src/libs/auth) to create the database roles used by our application. For this tutorial, the default suits us just fine, we only need to differentiate between logged-in users (webuser) and the rest (anonymous). Although you can use specific roles to mean specific users (alice, bob) you'll probably use database roles to mean groups of users (admin, employee, customer). Notice how in the [`users`](https://github.com/subzerocloud/subzero-starter-kit/blob/master/db/src/data/schema.sql#L11) table we have a column `user_type` and its value is `webuser`, this is the value that will also be reflected in the JWT payload under the key `role`.

Now about the mystery `JWT` token. I was only able to generate a valid token (that does not expire) because I knew the secret with which PostgREST is started (which is `reallyreallyreallyreallyverysafe` and is defined in `.env`). Head over to [jwt.io](http://jwt.io) site and paste in the token

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoxLCJyb2xlIjoid2VidXNlciJ9.uSsS2cukBlM6QXe4Y0H90fsdkJSGcle9b7p\_kMV1Ymk

You will see it's payload decoded

{
  "user\_id": 1,
  "role": "webuser"
}

We use the information in the JWT payload to see who is making the request. It's important to understand that anyone can look at the JWT payload, it's just a base64 encoded string (so don't put sensitive data inside it) but they can not alter its payload without knowing the secret.

### Row Level Security[¶](#row-level-security "Permanent link")

We left things in the state of every logged-in user having access to all the data rows in the main tables (through views).

So let's fix that. For this, we'll use a feature introduced in PostgreSQL 9.5 called "Row Level Security", RLS for short.

Before we start defining policies for each of the tables, because the users are accessing the data in the tables through views in the `api` schema, we will need to explicitly specify the owner of the view. This is needed because when someone accesses a table with policies through a view, the `current_user` switches from the (database) logged-in role to the role of the view owner.

Add these lines at the end of `db/src/api/views_and_procedures.sql`

\-- ...
alter view clients owner to api;
alter view projects owner to api;
alter view tasks owner to api;
alter view comments owner to api;

Just as with the `webuser`, `api` is a simple role defined [here](https://github.com/subzerocloud/subzero-starter-kit/blob/master/db/src/api/schema.sql#L8)

Now we get to the interesting part, defining the policies for each table and we'll use the information contained in the JWT token to restrict the rows a user can access.

Let's open the `db/authorization/privileges.sql` file and add the following policies

\-- ...
set search\_path \= data, public;

alter table client enable row level security;
grant select, insert, update, delete on client to api;
create policy access\_own\_rows on client to api
using ( request.user\_role() \= 'webuser' and request.user\_id() \= user\_id );

alter table project enable row level security;
grant select, insert, update, delete on project to api;
create policy access\_own\_rows on project to api
using ( request.user\_role() \= 'webuser' and request.user\_id() \= user\_id );

alter table task enable row level security;
grant select, insert, update, delete on task to api;
create policy access\_own\_rows on task to api
using ( request.user\_role() \= 'webuser' and request.user\_id() \= user\_id );

alter table project\_comment enable row level security;
grant select, insert, update, delete on project\_comment to api;
create policy access\_own\_rows on project\_comment to api
using ( request.user\_role() \= 'webuser' and request.user\_id() \= user\_id );

alter table task\_comment enable row level security;
grant select, insert, update, delete on task\_comment to api;
create policy access\_own\_rows on task\_comment to api
using ( request.user\_role() \= 'webuser' and request.user\_id() \= user\_id );

Now that we have the policies set, let's try the last request again

curl \-H "Authorization: Bearer $JWT\_TOKEN" http://localhost:8080/rest/clients?select\=id,name

And the result is

\[{"id":1,"name":"Apple"},{"id":2,"name":"Microsoft"}\]

Much better!

Now, for this to work when new data is inserted, we need to give a default value for the `user_id` column in each table.

Have this column definition for all tables in the `data` schema look like this

user\_id      int not null references "user"(id) default request.user\_id(),

### Authentication[¶](#authentication "Permanent link")

Ok, so now everything is set up, everyone can get to his data and change it, but how do they get their foot in the door, how do they signup, how do they log in. This is where the stored procedures come into play, and by that I mean any stored procedure defined in the `api` schema. Whenever you have to implement something that can not be expressed using a single query, i.e. it's a sequential process, you would use a stored procedure like this. There is always a temptation to design your whole API as a set of stored procedures, it seems you have greater/more targeted control but you will be putting yourself in the corner and you will have to write a new function for each new feature. Try sticking to views whenever possible and make them as general as possible then let the clients select from them with additional filters. If you can, write those functions in PL/pgSQL (but you can also use other languages), it does not have a shiny syntax and it can be weird in places but because of it's close relation to the database and SQL, it can express some problems very neatly and concise, even if you've never looked at this language before, you'll get its meaning easely.

For the authentication process we use the functions from the [auth](https://github.com/subzerocloud/subzero-starter-kit/blob/master/db/src/libs/auth) lib, but you can write your own if you want to. Check out the [definition](https://github.com/subzerocloud/subzero-starter-kit/blob/master/db/src/libs/auth/api/login.sql) of the login function. Our api schema is already [includding](https://github.com/subzerocloud/subzero-starter-kit/blob/master/db/src/api/schema.sql#L14) it so let's try and call it.

curl \\
-H "Content-Type: application/json" \\
-H "Accept: application/vnd.pgrst.object+json" \\
-d '{"email":"alice@email.com","password":"pass"}' \\
http://localhost:8080/rest/rpc/login

{
  "me":{"id":1,"name":"alice","email":"alice@email.com","role":"webuser"}
}

Note

If you are using [PostgREST Starter Kit](https://github.com/subzerocloud/postgrest-starter-kit) as the base of your project, the response body will also contain the JWT token. In [subZero Starter Kit](https://github.com/subzerocloud/subzero-starter-kit) the token is transparently transformed into the `SESSIONID` cookie by the [jwt\_session\_cookie](https://github.com/subzerocloud/subzero-starter-kit/blob/master/openresty/lualib/user_code/hooks.lua#L6) module to provide a more secure session storage mechanism on the client side.

Input Validation[¶](#input-validation "Permanent link")
-------------------------------------------------------

Back in the old days, the DB was the gatekeeper, it made sure who had access to what and most importantly it did not let any bad data get it (assuming the DBA knew what he was doing). It did not matter how you accessed the database and from what language, you could not write `asdf` in a field that was supposed to hold a phone number. Well, we are going to bring that back again, we are going to make databases great again :) and we are going to do that by using constraints. No more validators all over the place that check the user input (and that you always forget to implement in your new shiny client that accesses the database). By placing a constraint on a field, no one can bypass it, even if he really wants to.

You can read more about constraints [here](http://nathanmlong.com/2016/01/protect-your-data-with-postgresql-constraints/) but for our case, I'll just paste the additional lines you need to add to the table definitions and notice how nice the syntax is. You can use in constraint definitions any valid SQL expression that returns a `bool`, even custom functions defined by you.

Here are a few examples of constraints one might choose to set up

create table client (
  ...
  check (length(name)\>2 and length(name)<100),
  check (updated\_on is null or updated\_on \> created\_on)
);

create table project (
  ...
  check (length(name)\>2),
  check (updated\_on is null or updated\_on \> created\_on)
);

create table task (
  ...
  check (length(name)\>2),
  check (updated\_on is null or updated\_on \> created\_on)
);

create table task\_comment (
  ...
  check (length(body)\>2),
  check (updated\_on is null or updated\_on \> created\_on)
);

create table project\_comment (
  ...
  check (length(body)\>2),
  check (updated\_on is null or updated\_on \> created\_on)
);

Let's check if the rules are applied

curl \\
-H "Authorization: Bearer $JWT\_TOKEN" \\
-H "Content-Type: application/json" \\
-H 'Prefer: return=representation' \\
-d '{"name":"A"}'  \\
http://localhost:8080/rest/clients

{"hint":null,"details":null,"code":"42501","message":"permission denied for sequence client\_id\_seq"}

Woops :) again, a good thing :) the database is very specific and strict about access rights.

Let's fix that in `db/authorization/privileges.sql`, add the lines.

\-- ...
grant usage on sequence data.client\_id\_seq to webuser;
grant usage on sequence data.project\_id\_seq to webuser;
grant usage on sequence data.task\_id\_seq to webuser;
grant usage on sequence data.task\_comment\_id\_seq to webuser;
grant usage on sequence data.project\_comment\_id\_seq to webuser;

Now the request again

curl \\
-H "Authorization: Bearer $JWT\_TOKEN" \\
-H "Content-Type: application/json" \\
-H "Accept: application/vnd.pgrst.object+json" \\
-H 'Prefer: return=representation' \\
-d '{"name":"A"}'  \\
http://localhost:8080/rest/clients

{"hint":null,"details":null,"code":"23514","message":"new row for relation \\"client\\" violates check constraint \\"client\_name\_check\\""}

curl \\
-H "Authorization: Bearer $JWT\_TOKEN" \\
-H "Content-Type: application/json" \\
-H "Accept: application/vnd.pgrst.object+json" \\
-H 'Prefer: return=representation' \\
-d '{"name":"Uber"}'  \\
http://localhost:8080/rest/clients?select\=id,created\_on

{"id":4,"created\_on":"2017-07-18T12:13:47.202074+00:00"}

Mutations on complex views[¶](#mutations-on-complex-views "Permanent link")
---------------------------------------------------------------------------

A lot of the times your views will be mostly a "mirror" of your underlying tables, maybe you'll rename or exclude a few columns, have some computed ones. In such cases, they automatically become "updatable views", meaning PostgreSQL knows how to handle an insert/update/delete on the view. However, there are cases when you create views that are a bit more complex like we did with `comments`. In such cases, you have to help out the database and tell it how to handle mutations on the view.

First, we'll create our own function that knows how to route the data to the appropriate table. Since this function has to go in some schema, let's create one called `util`. We don't want to put it in the `api` schema since it will look like an endpoint that clients could call and we also don't want it in the `data` schema because it's a good idea to keep just table definitions there

\-- db/src/libs/util/schema.sql
drop schema if exists util cascade;
create schema util;
set search\_path \= util, public;

now create this file

\-- db/src/libs/util/mutation\_comments\_trigger.sql
create or replace function mutation\_comments\_trigger() returns trigger as $$
declare
    c record;
    parent\_type text;
begin
    if (tg\_op \= 'DELETE') then
        if old.parent\_type \= 'task' then
            delete from data.task\_comment where id \= old.id;
            if not found then return null; end if;
        elsif old.parent\_type \= 'project' then
            delete from data.project\_comment where id \= old.id;
            if not found then return null; end if;
        end if;
        return old;
    elsif (tg\_op \= 'UPDATE') then
        if (new.parent\_type \= 'task' or old.parent\_type \= 'task') then
            update data.task\_comment 
            set 
                body \= coalesce(new.body, old.body),
                task\_id \= coalesce(new.task\_id, old.task\_id)
            where id \= old.id
            returning \* into c;
            if not found then return null; end if;
            return (c.id, c.body, 'task'::text, c.task\_id, null::int, c.task\_id, c.created\_on, c.updated\_on);
        elsif (new.parent\_type \= 'project' or old.parent\_type \= 'project') then
            update data.project\_comment 
            set 
                body \= coalesce(new.body, old.body),
                project\_id \= coalesce(new.project\_id, old.project\_id)
            where id \= old.id
            returning \* into c;
            if not found then return null; end if;
            return (c.id, c.body, 'project'::text, c.project\_id, c.project\_id, null::int, c.created\_on, c.updated\_on);
        end if;
    elsif (tg\_op \= 'INSERT') then
        if new.parent\_type \= 'task' then
            insert into data.task\_comment (body, task\_id)
            values(new.body, new.task\_id)
            returning \* into c;
            return (c.id, c.body, 'task'::text, c.task\_id, null::int, c.task\_id, c.created\_on, c.updated\_on);
        elsif new.parent\_type \= 'project' then
            insert into data.project\_comment (body, project\_id)
            values(new.body, new.project\_id)
            returning \* into c;
            return (c.id, c.body, 'project'::text, c.project\_id, c.project\_id, null::int, c.created\_on, c.updated\_on);
        end if;

    end if;
    return null;
end;
$$ security definer language plpgsql;

Remember to include this function in the schema file

echo "\\ir mutation\_comments\_trigger.sql;" >> db/src/libs/util/schema.sql

Include the `util` schema as a dependency in the `db/src/init.sql` file.

\-- add this line just below the place where rabbitmq lib is included
\\ir libs/util/schema.sql

and the last step is to attach the function to the view as a trigger in `db/src/api/views_and_procedures.sql`

\-- ...
create trigger comments\_mutation
instead of insert or update or delete on comments
for each row execute procedure util.mutation\_comments\_trigger();

Now the view looks just like another table with which we can work as usual. A different way to handle this would be to expose distinct stored procedures like `insert_comment` `update_comment` or even more specific like `add_task_comment`.

Let's try an insert

curl -s -X POST \\
-H "Authorization: Bearer $JWT\_TOKEN" \\
-H "Content-Type: application/json" \\
-H "Accept: application/vnd.pgrst.object+json" \\
-H 'Prefer: return=representation' \\
-d '{"body": "Hi there!","parent\_type": "task","task\_id":1}'  \\
http://localhost:8080/rest/comments| \\
python -mjson.tool

the output will look like

{
  "id":3,
  "body":"Hi there!",
  "parent\_type":"task",
  "parent\_id":1,
  "project\_id":null,
  "task\_id":1,
  "created\_on":"2017-08-29T02:04:29.35094+00:00",
  "updated\_on":null
}

and an update now

curl -s -X PATCH \\
-H "Authorization: Bearer $JWT\_TOKEN" \\
-H "Content-Type: application/json" \\
-H "Accept: application/vnd.pgrst.object+json" \\
-H 'Prefer: return=representation' \\
-d '{"body":"This is going to be awesome!"}'  \\
"http://localhost:8080/rest/comments?id=eq.1&parent\_type=eq.project" | \\
python -mjson.tool

the output:

{
  "id":1,
  "body":"This is going to be awesome!",
  "parent\_type":"project",
  "parent\_id":1,
  "project\_id":1,
  "task\_id":null,
  "created\_on":"2017-07-18T11:31:12+00:00",
  "updated\_on":null
}

Progress[¶](#progress "Permanent link")
---------------------------------------

Let's stop for a second and see how much we've accomplished so far.

curl -s -G \\
-H "Authorization: Bearer $JWT\_TOKEN" \\
http://localhost:8080/rest/clients \\
--data-urlencode select\="id,name,projects(id,name,comments(body),tasks(id,name,comments(body)))" | \\
python -mjson.tool

What you see above is a request that will return all the clients and their projects and for each project, we ask also for tasks and comments. All that in a single request. At the same time, the access to those entities is checked and only the right rows are returned.

Let's see how much code we had to write for that.

$ $ cloc --include-lang=SQL db/src/api/views\_and\_procedures.sql db/src/data/tables.sql db/src/authorization/privileges.sql db/src/libs/util/
       5 text files.
       5 unique files.                              
       0 files ignored.

http://cloc.sourceforge.net v 1.64  T=0.03 s (146.2 files/s, 8276.3 lines/s)
-------------------------------------------------------------------------------
Language                     files          blank        comment           code
-------------------------------------------------------------------------------
SQL                              5             47             23            213
-------------------------------------------------------------------------------
SUM:                             5             47             23            213
-------------------------------------------------------------------------------

In just about 200 LOC, out of which about half are table definitions, we implemented a REST API. We have an authorization mechanism and we can make sure each user sees only what he is supposed to see (his data). We've decoupled our `api` from our underlying `data` model by using a separate schema that has only views and stored procedures in it. We do our user input validation using constraints and for any process that is more complex and takes a few steps to complete (login/signup) we can use stored procedures in all the languages supported by PostgreSQL. To do all this we did not have to write a single line of imperative logic, we only defined our data and the rules controlling the access.

And that is all, we have the core of our API ready, the part that deals strictly with the data inside our database. It's time to take a break from all that coding and click around a bit and check out that API
