

Using the API
=============

REST[¶](#rest "Permanent link")
-------------------------------

At the heart of subZero sits [PostgREST](http://postgrest.com) and as such, the REST interface is actually the same as the one exposed by PostgREST. What subZero does is put a proxy in front of PostgREST in order to be able to have control of the url prefix of the api, cache responses and provide a mechanism to restrict which columns are available for filtering plus a lot of other usefull features that are not the target of PostgREST. Other then that, anything describe in the PostgREST documentation, you can trust it's available in subZero REST interface. Rather then maintaining duplicate documentation here, you can consult the [exelent docs](https://postgrest.com/en/latest/) that PostgREST provides. The only thing to remember is that whenever you see a url like `/people`, it's equivalent in a default subZero installation is `/rest/people` (but you can change that if you like)

Let's just check a few examples:

**Get the projects of a specific client and all the active tasks for them**

curl -s -G -X GET \\
-H "Authorization: Bearer $JWT\_TOKEN" \\
http://localhost:8080/rest/projects \\
--data-urlencode select\="id,name,tasks(id,name)" \\
--data-urlencode client\_id\="eq.1" \\
--data-urlencode tasks.completed\="eq.false"  | \\
python -mjson.tool

**Add a client and return it's new id and created\_on field**

curl -s -X POST \\
-H "Authorization: Bearer $JWT\_TOKEN" \\
-H "Content-Type: application/json" \\
-H "Accept: application/vnd.pgrst.object+json" \\
-H 'Prefer: return=representation' \\
-d '{"name":"Google","address":"Mountain View, California, United States"}'  \\
http://localhost:8080/rest/clients?select\=id,created\_on | \\
python -mjson.tool

**Update a projects's name**

curl -s -X PATCH \\
-H "Authorization: Bearer $JWT\_TOKEN" \\
-H "Content-Type: application/json" \\
-H "Accept: application/vnd.pgrst.object+json" \\
-H 'Prefer: return=representation' \\
-d '{"name":"Updated name"}'  \\
"http://localhost:8080/rest/projects?select=id,name,created\_on,updated\_on&id=eq.1" | \\
python -mjson.tool

You probably just noticed that after the operation above, `updated_on` field was still `null`. We forgot to give it a value. Easy fix, add this to `db/src/data/tables.sql` file

\-- ...
create or replace function set\_updated\_on() returns trigger as $$
begin
  new.updated\_on \= now();
  return new;
end
$$ language plpgsql;

create trigger client\_set\_updated\_on
before update on "client"
for each row execute procedure set\_updated\_on();

create trigger project\_set\_updated\_on
before update on "project"
for each row execute procedure set\_updated\_on();

create trigger task\_set\_updated\_on
before update on "task"
for each row execute procedure set\_updated\_on();

create trigger task\_comment\_set\_updated\_on
before update on "task\_comment"
for each row execute procedure set\_updated\_on();

create trigger project\_comment\_set\_updated\_on
before update on "project\_comment"
for each row execute procedure set\_updated\_on();

If you try that update again, you will see the property change each time.

**Delete a task**

curl -s -X DELETE \\
-H "Authorization: Bearer $JWT\_TOKEN" \\
-H 'Prefer: return=representation' \\
"http://localhost:8080/rest/tasks?select=id,created\_on&id=eq.2" | \\
python -mjson.tool

Info

If you try to delete task #1, you will get an error telling you:

`Key is still referenced from table "task_comment"`

This is another example of a safety net the database gives you when you leverage its features, it will not let you corrupt your data by accident. We will have to be specific using `ON DELETE` what happens to related comments when a task is deleted. It might seem annoying at first but this strictness will save you a lot of headake in the long run.

GraphQL[¶](#graphql "Permanent link")
-------------------------------------

Some of you might have come here for GraphQL specifically and all we've shown you so far was SQL and REST. So without any further ado, open this address in your browser [http://localhost:8080/graphiql/](http://localhost:8080/graphiql/)

This is a nice IDE available in development mode to interact with your API. Use the auto-complete and the automatically generated documentation on the right to see what you can do with this api and by the way, the GraphQL endpoint for the api is `http://localhost:8080/graphql/simple/` and `http://localhost:8080/graphql/relay/` depending on the flavour you prefer.

Let's try and run a few queries:

**Login**, the browser will get back a session cookie that will allow us to execute other requests in an authenticated state

{
  login(email: "alice@email.com", password: "pass"){
    me
  }
}

**Request a specific object by key**

{
  task(id: 1){
    name
  }
}

**Filtering results by columns on each level**

{
  projects(where: {name: {ilike: "\*3\*"}}) {
    id
    name
    tasks(where: {completed: {eq: false}}){
      id
      name
    }
  }
}

**Requesting multiple levels of data**

{
  clients{
    id 
    name
    projects {
      id
      name
      tasks{
        id
        name
        comments {
          body
          created\_on
        }
      }
    }
  }
}

**Insert one client**

mutation {
  insert {
    client(input: {name: "Apple", address: "Cupertino, California, United States"}) {
      id
      created\_on
    }
  }
}

**Insert multiple projects at the same time**

mutation {
  insert {
    projects(input: \[
      {name: "Project 1", client\_id: 1},
      {name: "Project 2", client\_id: 1}
    \]){
      id
      created\_on
    }
  }
}

Info

Having the `input` parameter directly in the query is a bit ugly, GraphQL supports variables and you can separate the input from the query in your frontend code

**Update a client**

mutation {
  update {
    client(id: 1, input: {name: "Updated name"}){
      id
      name
      updated\_on
    }
  }
}

If you have `LivePage` extension in chrome, when iterating over your API, just like in REST, all you need to do is edit the files and save them and by the time you `Alt+TAB` to the browser, the page is refreshed and ready for you to (re)execute your GraphQL query.

Info

If you are familiar with the GraphQL ecosystem, you've probably figured out by now that this structure for the API will not work with [Relay](https://facebook.github.io/relay/). subZero is capable of generating a compatible API but for the purpose of this tutorial we used this simple version to better see the relation between the GraphQL types and the entities in the `api` schema without the mental overhead of such concepts as `viewer`, `connections`, `edges`
