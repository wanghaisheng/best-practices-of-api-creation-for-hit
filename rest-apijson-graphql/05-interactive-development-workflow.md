
Iterative Development Workflow
==============================

Overview[¶](#overview "Permanent link")
---------------------------------------

One of the reasons developer avoid putting any logic in the database is because it's a pain to move that logic from a file (stored in git) to the environment (the database) where it is executed. In many ways it's similar to the workflow you have when developing in a compile language like C/Java/Go where there is an additional step, compiling, before you can see the code in action. But with databases it's a bit worse because it's not as easy to move that code as a one line command to compile a C program. This is the main reason we developed the [subzero-cli](https://github.com/subzerocloud/subzero-cli). The goal is to make the development workflow for writing code that lives in the database (tables/views/stored procedures/...) as close as possible to the workflow of a Ruby/PHP/Node developer. We want to be able to just save a (sql) file and have that logic immediately running in the database, ready for you to execute. While moving SQL code from files to the database is a core feature, `subzero-cli` does a lot more, it will also do the same thing for nginx configs and lua code (running in nginx). It will also give you a nice interface to look at the logs of each component of the stack and see the result of a HTTP call to the api in addition to creating and managing your database migrations.

Check out how the whole process looks (although you don't see in the gif the process of saving the file).

![starter kit](../images/postgrest-starter-kit.gif)

Install[¶](#install "Permanent link")
-------------------------------------

docker pull subzerocloud/subzero-cli-tools
npm install -g subzero-cli
subzero --help \# check it was installed

Workflow[¶](#workflow "Permanent link")
---------------------------------------

In the root of your project (which is a clean version of the starter kit) execute this

docker-compose -d up db postgrest openresty
subzero dashboard

Notice how we chose to bring up only a subset of the stack components since we won't be using rabbitmq for now. In another terminal window, run the following command

curl http://localhost:8080/rest/todos?select\=id,todo

\# result
\[{"id":1,"todo":"item\_1"},{"id":3,"todo":"item\_3"},{"id":6,"todo":"item\_6"}\]

Notice the result of this in each dashboard window/tab (OpenResty/PostgREST/PostgreSQL) each showing a log line of the event they registered. Probably the most interesting window for now will be PostgreSQL where you can see the actual queries that were executed for that REST call, which in this case look something like this (with a bit of formatting compared to the real thing)

LOG:  execute 0: BEGIN ISOLATION LEVEL READ COMMITTED READ ONLY                                                                                                                 
LOG:  statement: set local role 'anonymous';set local "request.jwt.claim.role" \= 'anonymous';set local "request.header.host" \= 'postgrest';set local "request.header.user-agent" \= 'curl/7.43.0';set local "request.header.accept" \= '\*/\*';                                                                                                                    │
LOG:  execute <unnamed\>:                                                                                                                                                        
          WITH pg\_source AS (
              SELECT  "api"."todos"."id", "api"."todos"."todo" FROM  "api"."todos"
          )
          SELECT 
            null AS total\_result\_set, 
            pg\_catalog.count(\_postgrest\_t) AS page\_total, 
            array\[\]::text\[\] AS header, 
            coalesce(array\_to\_json(array\_agg(row\_to\_json(\_postgrest\_t))), '\[\]')::character varying AS body
          FROM ( SELECT \* FROM pg\_source) \_postgrest\_t
LOG:  execute 1: COMMIT

Now we'll change the `todos` view. Change [this line](https://github.com/subzerocloud/subzero-starter-kit/blob/master/db/src/api/todos.sql#L9)

#before
select id, todo, private, (owner\_id \= request.user\_id()) as mine from data.todo;

#after
select ('#' || id::text) as id, ('do this: ' || todo) as todo, private, (owner\_id \= request.user\_id()) as mine from data.todo;

Save the file, then run the previous curl command

curl http://localhost:8080/rest/todos?select\=id,todo

#result
\[{"id":"#1","todo":"do this: item\_1"},{"id":"#3","todo":"do this: item\_3"},{"id":"#6","todo":"do this: item\_6"}\]

And that's the gist of it. You save your code, be that SQL/Lua/Nginx and it's immediately running live in your dev stack.
