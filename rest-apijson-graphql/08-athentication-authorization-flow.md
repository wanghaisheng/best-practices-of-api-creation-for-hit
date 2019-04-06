
Athentication Authorization Flow
================================

### Json Web Tokens[¶](#json-web-tokens "Permanent link")

PostgREST uses [Json Web Tokens](https://jwt.io) to authenticate users. A JWT looks like this

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJyb2xlIjoid2VidXNlciIsInVzZXJfaWQiOjF9.\_Y-0Bk5emC7KOSNQaK-zBHFUW0wt055cj8TOi74pj1o

Each JWT has a payload, the payload is visible to the client that received the token (so don't put secret stuff there), but he can not change the payload (generate his JWT) without knowing the secret key.

The payload for the above token is

{
  "role": "webuser",
  "user\_id": 1
}

PostgREST specifically cares about the `role` key. This property decides what specific database users we need to switch into before executing the request (so that the authorization policies we defined take effect). PostgREST will also check the `exp` field which should be a Unix timestamp to see if the token is still valid. Every other property is just communicated to the database, PostgREST does not take any action based on that. One such property is `user_id`. It's a custom field for the starter kit. Based on this property from the JWT, we have defined some RLS policies which decide what rows a user is allowed to see.

PostgREST does not have any logic inside it to generate JWT tokens (it used to) because they can be created by any other script or even by a 3rd party service like [Auth0](https://auth0.com/). In this starter kit, we've chosen the most simple implementation which will suffice for most use cases. We generate the JWT token directly in the database.

Ok, so now the client has this token after calling the login function. Next time he wants to make a request, the can communicate his identity to the API by sending the following header

### Authorization[¶](#authorization "Permanent link")

Authorization: Bearer JWT\_TOKEN\_GOES\_HERE

If you'd prefer to use cookies instead of headers for security reasons, it's entirely possible to have a short script running in OpenResty that does the transformation on the fly from a cookie value to a header without the need for anything custom in the client or PostgREST. We use this method in [subZero](https://subzero.cloud) to signal the authenticated state with cookies.

When PostgREST receives such a request, the first thing it will do is check if the token is valid using the secret key, once that is done, it will check if it did not expire by looking at the `exp` key in the payload. If all that is ok, we move on to the next stage.

### A Sample Transaction[¶](#a-sample-transaction "Permanent link")

The first thing to do is open a transaction. At this point, if the JWT payload contains the `role` key, PostgREST will switch to that, then it will set different context variables like request headers and other keys from the JWT payload. Only when the above is done, PostgREST will execute the main query.

A complete request looks like this (with just a little bit of formatting)

 \-- transaction start
BEGIN ISOLATION LEVEL READ COMMITTED READ ONLY;

\-- switch to the appropriate role
set local role 'webuser';

\-- set the value of all jwt keys
set local "request.jwt.claim.role" \= 'webuser';
set local "request.jwt.claim.user\_id" \= '1';

\-- set the value of all heaaders
set local "request.header.host" \= 'postgrest';
set local "request.header.user-agent" \= 'node-superagent/3.5.2';
set local "request.header.accept" \= 'application/vnd.pgrst.object+json';

\-- main query starts here
WITH pg\_source AS (
 \-- this is the essence of the query, the stuff around it is mostly to json encode the result
 SELECT  "api"."items"."id", "api"."items"."name"
 FROM  "api"."items"  
 WHERE  "api"."items"."id" \= '1'::unknown
) 
\-- this is the wrapping query
SELECT 
 null AS total\_result\_set, 
 pg\_catalog.count(\_postgrest\_t) AS page\_total, 
 array\[\]::text\[\] AS header, 
 coalesce(string\_agg(row\_to\_json(\_postgrest\_t)::text, ','), '')::character varying  AS body
FROM ( SELECT \* FROM pg\_source ) \_postgrest\_t;

\-- close the transaction
COMMIT;

Whenever you want to understand what's going on or debug the request, you take a look a the pg\_source CTE (Common Table Expression, the query inside the WITH statement) and see if it's doing what you think it should be doing.

### Mental Model[¶](#mental-model "Permanent link")

Now I will lay out a mental model of how to think about the steps the database uses to execute this query, in reality, it's a more efficient process.

When the database is executing that query, it will check is if `webuser` has the right to run SELECT against the `api.items` view and also check each column if it's available for querying. That's because you can have something like a table `employees` with columns `id, name, salary` and you can specify that `admin` has the right to view all columns but `employee` only `id` and `name`.

Once these checks pass, the database will try to execute the definition of the `api.items` view, which is basically `SELECT * FROM data.items`, with the privileges of the view owner (usually the user that created the view). It's all very strict :). In our case, we set the view owner to a user called `api`. If the check passes, the database will execute the `SELECT *` query.

Now to the `api` user, it looks like he will be getting all the rows from the `data.items` table but we are not done yet. Before PostgreSQL executes that query, it will check if that table has row level security enabled and there are policies that apply to the current user (the current user is the view owner `api` and not the original one that executed the top query `webuser`).

It just so happens that there are some policies, so instead of running the `SELECT *` query, the database will take their definition and glue them all together with the query, so what ends up being executed against `data.items` is something like this

SELECT \* FROM data.items \-- original view query
\-- conditions from policies
WHERE
 (request.user\_role() \= 'webuser' and request.user\_id() \= owner\_id) OR
 (private \= false)

Once we have the correct rows returned from the `data.items` table, they are returned to the view. At this point, the top conditions of the request are applied (`WHERE "api"."items"."id" = '1'::unknown`), and the api call will return the resulting rows (after encoding them to JSON).
