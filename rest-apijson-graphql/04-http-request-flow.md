

HTTP Request Flow
=================

![Rquest Flow](../images/request-flow.png "Request Flow")

### OpenResty[¶](#openresty "Permanent link")

The first component in that stack that an HTTP request reaches is OpenResty (Nginx + Lua). The main job of OpenResty is to route requests and give you the possibility to alter the shape of the request (and response on the way back) while it's in flight between the browser and PostgREST. Within Openresty, the request flows through a few subcomponents, all of this happens in memory and is very efficient.

Assuming the request is an API call, it first reaches the `/rest` location. The only thing this subcomponent does is to set the CORS headers if needed and then route the request to `/internal/rest`.

This is, however, the place where you would place additional custom configuration like rate limiting, access control and other checks you want to run before the request is sent for processing. You can do that by adding the custom configuration in a new file that is placed in `openresty/nginx/conf/includes/http/server/locations/rest/` directory.

Once request passes the first location, it ends up in `/internal/rest`. This is where the most work inside OpenResty happens. On the way to the database, withing this location we make a call to `hooks.on_rest_request()` Lua function.

You can find it's definition in `openresty/lualib/user_code/hooks.lua` and it's being called in `openresty/lualib/user_code/internal_rest_rewrite_phase.lua`. You can use this hook to execute complex logic against the current request.

For example you can check that a particular endpoint is called only by authenticated users (instead of letting it reach the database in anonymous state and waste a few CPU cycles) or you can make sure that you accept delete/update calls only to single rows using the primary key (because by default PostgREST allows you to delete multiple/all rows in a table). You can even call external systems or run queries against your DB if you need to.

After the request passes through the `/internal/rest` location it's handed to `upstream postgrest` subcomponent that is responsible for forwarding it to PostgREST. In the upstream configuration, it's possible to tune connection parameters between openresty and postgrest, for example, the number of concurrent connections to keep alive for better efficiency.

### PostgREST[¶](#postgrest "Permanent link")

Once the request leaves OpenResty, it reaches PostgREST. There is not much you can configure here not much to explain except this is the component that will translate the HTTP request to a SQL query and execute it against the database. This is not to say that it's a less important part of the stack, it's arguably the component that allows the possibility of this stack existing.

Each HTTP call that reaches PostgREST will result in SQL query that is executed within a transaction. Before running that query, PostgREST will set some context like the current authenticated users so that the database knows which policies to enforce and also set some database variables with some header values so that you can use them in your stored procedures/triggers/views if you need to have custom logic based on that (for example log the IP of each call).

### PostgreSQL[¶](#postgresql "Permanent link")

While PostgREST is the component that enables this architecture, the power-horse that does all the heavy lifting (or should I say power elephant) is PostgreSQL. The main query that PostgREST sent over will be against one of the entities in the `api` schema. Even before doing any work, the database will check if the current user even has rights to run that query and access the api entity.

Assuming he has those privileges set, the definition of the view (which is basically a query) in the api schema will be executed to request the data from the underlying source tables that live in the `data` schema. At this point, the Row Level Security (RLS) policies kick in and make sure the users see only the rows they are supposed to see and not the entire table. Although all the rows live in the same table, to the user it will seem as if he has his own table with his private rows.

Then those rows are handed over to the view and then the additional filters the original HTTP request sent will be applied.

After this explanation, you might think that this is very inefficient, all those stages between the API entity and the source tables. However, this is only a mental model, how to think about the way the query is executed in the database. This is not the way it actually happens. PostgreSQL query planner/optimizer will take all those steps, grants, query conditions, view condition, RLS rules, combine them all together and basically make one single query and execute it in one step, it's smart like that :).

Once the all the rows are in memory (again, mental model, not strictly true) the rows will be encoded as JSON and returned to PostgREST. PostgREST, will just take that reply from the database and forward it back to OpenResty.

On the way back to the client, the response will flow back mostly the same way the request came in, it will go through the upstream subcomponent then end up in `/internal/rest` location. We've defined a Lua hook here to give you the chance to interact with the response on the way back. You can make any call to an external system here, you can add additional headers to the response in the `hooks.before_rest_response()`. Also within that hook, you can decide if you want to even alter the response body (JSON payload).

This was the last step after you've had the chance to interact with the response, it's sent back to the client (the browser).

### RabbitMQ[¶](#rabbitmq "Permanent link")

We've said above that the Lua hooks are the place where you can add some custom logic. That code, however, is executed synchronously and that is exactly what you need probably but there are a lot of situations where the additional work you need to do can be executed asynchronously, like send an email, log an event in another system, implement live updates. For cases like this, it's better to use RabbitMQ. From the database you can generate an event, from a trigger or a stored procedure, that event ends up in RabbitMQ (with the help of pg-amqp-bridge) and from there you can route it in complex ways to multiple consumers which can be phones/browsers listening for live updates or separate worker scripts that do some work with the event.
