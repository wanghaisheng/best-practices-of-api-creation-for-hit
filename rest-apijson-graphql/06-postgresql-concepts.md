

PostgreSQL Concepts
===================

If you haven't figured out by now, we've tricked you into being a DBA and 90% of the work you'll do will be in the database, defining things there, as such, you need a solid understanding of concepts and how they are leveraged to create the API.

Your reference, in this case, will almost always start with [PostgreSQL Manual](https://www.postgresql.org/docs/current/static/index.html).

Here is a summary of the core concepts and how you can leverage them in subZero based projects. We assume here you have a basic understanding of SQL and are comfortable with basic SELECT/INSERT/UPDATE/DELETE/JOIN.

### Tables[¶](#tables "Permanent link")

Tables are the cornerstone of all databases, and this is where your data is stored, and you will define your data model with tables. Start with [Table Basics](https://www.postgresql.org/docs/current/static/ddl-basics.html), [Data Types](https://www.postgresql.org/docs/current/static/datatype.html) and [CREATE TABLE](https://www.postgresql.org/docs/current/static/sql-createtable.html) statement.

### Constraints[¶](#constraints "Permanent link")

Once you have your data model defined with tables, you need to make sure the data in those tables follows some rules, is clean and consistent. [Constraints](https://www.postgresql.org/docs/current/static/ddl-constraints.html) are an excellent way to accomplish just that. If a constraint is set on a table, no one can insert bad data, no matter the privileges. Constraints are not just making sure a value is an int or a string, they can be much more elaborate and useful. Check out the article [Protect Your Data with PostgreSQL Constraints](http://nathanmlong.com/2016/01/protect-your-data-with-postgresql-constraints/) and [PostgreSQL Domain Integrity In Depth](https://begriffs.com/posts/2017-10-21-sql-domain-integrity.html)

### Triggers[¶](#triggers "Permanent link")

In some situations, whenever some data is changed in a table, you also need to reflect that change in another table, or maybe you need to send out event to notify other components in the stack about this change (real time updates, sending emails). These are tasks that can be solved by [triggers](https://www.postgresql.org/docs/current/static/triggers.html). They allow you to do additional work whenever someone executes and INSERT/UPDATE/DELETE (and some other events in addition to that). Writing triggers is also the place where you'll first encounter PostgreSQL procedural languages for (database) [server programming](https://www.postgresql.org/docs/current/static/server-programming.html).

### Roles, Grants, and Row Level Security[¶](#roles-grants-and-row-level-security "Permanent link")

The database has a rich role system able to express concepts such as users and groups. When leveraged properly, you can define complex authorization policies with the granularity down to a cell (intersection of a column with a row). Start with an [overview](https://www.postgresql.org/docs/role/static/user-manag.html), then check out [CREATE ROLE](https://www.postgresql.org/docs/current/static/sql-createrole.html) and [GRANT](https://www.postgresql.org/docs/current/static/sql-grant.html) statements. Once a user has access to a particular set of tables and columns in the database, you can restrict his access even more to only a subset of rows within those tables using a concept called [Row Level Security](https://www.postgresql.org/docs/current/static/ddl-rowsecurity.html). With the above explanation it might seem that you will need your application users also to be defined as database users, and while you can do that if you have a limited number of users accessing the application, there is a particular method you can use to avoid having to do that. You can still have your application users in a table like you are used to and be able to define database level access policies for them. Check out [this](https://blog.2ndquadrant.com/application-users-vs-row-level-security/) article that explains the method.

### Views[¶](#views "Permanent link")

One [common objection](https://news.ycombinator.com/item?id=9927771#9928436) to PostgREST in discussions is that it's a bad practice to couple your API to your data model. While on its own that statement makes sense, it's ignoring the fact that the database is capable of defining abstractions through [views](https://www.postgresql.org/docs/current/static/sql-createview.html). This method is a powerful way to decouple your data model from your API. When creating PostgREST based APIs, you will always expose schemas containing only views (and stored procedures), even if those views, in the beginning, are only "mirrors" of the source tables. You will have all the flexibility you need like hiding internal columns, renaming some, or even add some new columns which are computed in some way. If those views are not some aggregates, PostgreSQL is smart enough to make them [updatable](https://www.postgresql.org/docs/current/static/sql-createview.html). You will be able to execute INSERT/UPDATE/DELETE against them, and that will be routed to the underlying table, thus removing the need for you to write custom triggers in most of the cases.

### Stored Procedures[¶](#stored-procedures "Permanent link")

In some particular situations, there are processes that you can not define in a single query, most commonly because you need to change and analyze multiple tables before you can decide what to do. This type of complexity can be handled by [stored procedures](https://www.postgresql.org/docs/current/static/server-programming.html). When writing functions, try first to see if it's possible to implement it in [SQL](https://www.postgresql.org/docs/current/static/xfunc-sql.html) because PostgreSQL might be able to extract that logic and inline it when running the query (making it faster). If the logic is more complex, you can choose whatever language feels more comfortable (but give [PL/pgSQL](https://www.postgresql.org/docs/current/static/plpgsql.html) a good look, it's a bit faster than the others and good enough for writing a few reasonably complex functions). When you need extreme performance, know that you can even drop [down to C](https://www.postgresql.org/docs/current/static/xfunc-c.html).
