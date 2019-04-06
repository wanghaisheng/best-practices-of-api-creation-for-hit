

PostgREST Crash Course
======================

Assuming you have the stack up and running, here are the basic API calls you can make. For more in-depth reference check PostgREST [tutorials](https://postgrest.com/en/stable/tutorials/tut0.html) and the [api reference](https://postgrest.com/en/stable/api.html)

Note

By default in the starter kit, we have a URI prefix `/rest` compared to the PostgREST docs

Get one item

curl http://localhost:8080/rest/items/1

Get a filtered list of items

curl http://localhost:8080/rest/items?id\=gt.1

Specify the list of columns to return

curl http://localhost:8080/rest/items?id\=eq.1&select\=id,name

Embed related entities

curl http://localhost:8080/rest/items?id\=eq.1&select\=id,name,subitems(id,name)

Apply filters to the embedded items (the filters work on the second level, i.e., the list of items returned won't be affected by filters used on subitems)

curl http://localhost:8080/rest/items?id\=eq.1&select\=id,name,subitems(id,name)&subitems.name\=like.%subitem%

Insert one item and return it's id

curl -s -X POST \\
-H "Content-Type: application/json" \\
-H 'Prefer: return=representation' \\
-d '{"name":"New Item"}'  \\
http://localhost:8080/rest/items?select\=id

Usually, requests will need to be authenticated, so you'll also need to send the required header

\-H "Authorization: Bearer $JWT\_TOKEN" 

Update one item (as an authenticated user)

curl -s -X PATCH \\
-H "Authorization: Bearer $JWT\_TOKEN" \\
-H "Content-Type: application/json" \\
-H 'Prefer: return=representation' \\
-d '{"name":"Updated name"}'  \\
"http://localhost:8080/rest/items/1?select=id,name"

Delete one item (and get back its details before deleting it)

curl -s -X DELETE \\
-H "Authorization: Bearer $JWT\_TOKEN" \\
-H 'Prefer: return=representation' \\
"http://localhost:8080/rest/tasks?id=eq.1&select=id,name"

Notice how this time we used `id=eq.1` instead of `/items/1`, they are equivalent.
