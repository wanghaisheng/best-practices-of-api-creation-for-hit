
Beyond Data
===========

Overview[¶](#overview "Permanent link")
---------------------------------------

Most of us are used to building backend systems in a specific way. No matter the stack used, you usually have a web server in front that besides serving static content, routes all the requests to a collection of scripts that implement all the logic in the system (probably using some kind of framework to organize them). At the bottom of the stack sits the database that is only responsible for storing the data. This of course works but is not the only way to build systems and not necessarily the best way. The edges of this system (web server/database) are very powerful tools with a ton of functionality embedded in them and when properly leveraged, can result in a dramatic reduction of code complexity, performance gains and an increase in iteration speed. Just because your logic is in a single language/place does not mean the system is _[simpler](https://www.youtube.com/watch?v=rI8tNMsozo0)_. This starter kit takes a holistic approach when implementing an API and tries to leverage all features of the underlying components to achieve a particular goal.

Try to think of these components, not as separate processes (or even worse, distinct hardware) within your system but rather think of them as modules in your code. Just like in a traditional system control flows through a bunch of modules, an API request flows through these components and at each step you can take certain actions based on the current request. Just like in a traditional stack you would not implement caching in your auth module, or your "active record" module, it's the same here, you do not implement caching in the database but in front of it, even before the request hits the database. You do not send emails from within a stored procedure in the database, you send the email with a separate system that is only triggered by a database event.

Restricting filtering capabilities[¶](#restricting-filtering-capabilities "Permanent link")
-------------------------------------------------------------------------------------------

By default, subZero will allow the client to filter by all columns and different operators. When your tables are small and you do not have a lot of users, this is not a problem but it can become one if you allow filtering on a text column using `like` operator for a table that has hundreds of thousands of rows, especially if you do not have an index on that column. As a general rule, you should not allow filtering by a column if that column is not part of an index. Let's see how we can accomplish this seemingly complicated task.

Open `openresty/lualib/user_code/hooks.lua` file and add a function that will be called before each request (add it at the top of the file)

local function check\_filters()
   print 'I will be making sure no one does anything fishy'
end
\-- ...

and make sure to call it in `on_rest_request` function.

In your dev console, switch to the OpenResty tab and observe the result of the following request

curl -s -G -X GET \\
-H "Authorization: Bearer $JWT\_TOKEN" \\
"http://localhost:8080/rest/projects?name=like.\*OS" \\
| python -mjson.tool

You should see a few lines similar to these

2017/07/19 09:04:23 \[notice\] 53#53: \*109 \[lua\] hooks.lua:2: check\_filters(): I will be making sure no one does anything fishy, client: 172.18.0.1, server: \_, request: "GET  /rest/projects?name=like.\*OS HTTP/1.1", host: "localhost:8080"
2017/07/19 09:04:23 \[info\] 53#53: \*109 client 172.18.0.1 closed keepalive connection
\[19/Jul/2017:09:04:23 +0000\] "GET /rest/projects?name=like.\*OS HTTP/1.1" 200 204 "0.046 ms"

Now that we know our function is being called, let's see how we can accomplish our task. The best source for writing lua scripts in the context of OpenResty is the github repo README of the [lua-nginx-module](https://github.com/openresty/lua-nginx-module)

Change the function `check_filters` to this definition

local function check\_filters()
    local blacklist \= {
        projects\_name\_like \= true
    }
    local table \= ngx.var.uri:gsub('/internal/rest/', '')

    local args \= ngx.req.get\_uri\_args()
    for key, val in pairs(args) do
        local column \= key;
        local operator, value \= val:match("(\[^.\]+)%.(.\*)")
        if operator and blacklist\[ table .. '\_' .. column .. '\_' .. operator \] then
            ngx.status \= ngx.HTTP\_BAD\_REQUEST 
            ngx.say('filtering by ' .. column .. '/' .. operator .. ' is not allowed')
            ngx.exit(ngx.HTTP\_OK)
        end
    end
end

If we run this request now

curl -i -s -G -X GET \\
-H "Authorization: Bearer $JWT\_TOKEN" \\
"http://localhost:8080/rest/projects?name=like.\*OS"

The result will be

HTTP/1.1 400 Bad Request
Server: openresty
Date: Wed, 19 Jul 2017 09:43:47 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
X-Frame-Options: SAMEORIGIN
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Content-Location: /rest
Request-Time: 0.000

filtering by name/like is not allowed

This function is not perfect yet, the pattern matching could use a little more care but you get the idea how a few Lua lines in the OpenResty context can be a powerful thing when you want to change or alter the incoming request before it reaches the database.

Caching responses[¶](#caching-responses "Permanent link")
---------------------------------------------------------

All the components in a subZero system are fast and in most cases you don't need to worry about the load on your system and caching, especially if you were careful about restricting filtering capabilities in the section above. Don't go fixing problems you do not have. There are however cases when you have some entities, requests to which can and should be cached. Usually they will be views or stored procedures that implement a complex report that takes some time to generate. For this subZero exposes two lua hooks that allow you to fully control the caching subsystem. The functions are `get_cache_key` and `get_cache_tags`, the first one allows you to say "I want this response cached" and the second one allows you to say "I want cached responses with these tags to be invalidated". Let's suppose we have our `clients` view and it does not change that much and we were asked to introduce a property for each client called `profitability` which we implemented using a computed column and it's a time consuming computation. We would like to cache the responses for the clients endpoint to reduce the the number of times that complex `profitability` function is called.

The first thing to do is to enable the cache subsystem

\-- .env
...
DISABLE\_CACHE=0
...

Let's see how the functions might look.

\-- lua/user\_module.lua
local function get\_cache\_key(ngx\_vars, uri\_args, headers, get\_ast)
    local endpoint \= ngx\_vars.uri:gsub('/internal/rest/', '')
    if endpoint \== 'clients' then
        local sess\_id \= ngx\_vars.http\_authorization
        local key\_parts \= { sess\_id , ngx\_vars.uri, ngx\_vars.args or ''}
        local key \= table.concat(key\_parts,':')
        local ttl \= 60 \-- seconds
        local tags \= {sess\_id ..'\_clients'}
        return key, ttl, tags
    end
end

local function get\_cache\_tags(ngx\_vars, uri\_args, headers, get\_ast)
    local sess\_id \= ngx\_vars.http\_authorization
    local endpoint \= ngx\_vars.uri:gsub('/internal/rest/', '')
    return {endpoint .. '\_' ..endpoint}
end

Whenever there is a request to the clients endpoint, we compute a cache key for the requests. This key can be any string that uniquely identifies this request type. Notice we included the value of the `Authorization` header which identifies the current user so that we do not server a cache for one user to a different one. We also attach a tag to this cache instance so whenever a new client is inserted or a client is updated, we can invalidate it.

Try running this request a few times and see the cache working

curl -i \\
-H "Authorization: Bearer $JWT\_TOKEN" \\
http://localhost:8080/rest/clients

First request

...
Cache-Engine: "nginx"
Cache-Status: MISS
Cache-Key: 6481abcee926439946f1e6f47e45a1b5
Cache-TTL: 60
Method: GET
Request-Time: 0.009
...

Second request

...
Cache-Engine: "nginx"
Cache-Status: HIT
Cache-Key: 6481abcee926439946f1e6f47e45a1b5
Cache-TTL: 60
Method: GET
Request-Time: 0.000
...

Note

In development mode, unless you are working specifically on the caching logic, you want to have the cache disabled so that each request goes through.

Reacting to database events[¶](#reacting-to-database-events "Permanent link")
-----------------------------------------------------------------------------

The dirty secret: We’re all just building CRUD apps.

[@iamdevloper](https://twitter.com/iamdevloper/status/455409190505562112)

Whether we admit it or not, most of the time, 90% of what an API does is CRUD, enforce access rights and validate input, and that is OK :)! Up to this point, we've shown how you can do all that. If it mostly concerns the actual data, PostgreSQL is very flexible and allows you to easily interact with it, PostgREST provides the bridge from HTTP to SQL and OpenResty acts as a router in front of everything and it gives you the possibility to inspect and alter HTTP requests and responses that are in-flight.

However, even the most simple APIs usually have a couple of "paths" that need to interact with some other system and not only the specific data stored in the database. A classic example is sending an email on signup. The first thought is usually that you will have to do it inside a database trigger and actually you [can do that](http://brandolabs.com/pgmail), send an actual email. This shows just how versatile Postgres is. But you shouldn't, talking to remote systems is slow and you will be blocking for a good amount of time one database connection/process. Unless you are building a system with only a handful of users, avoid talking to remote systems from functions running in the database. Another possible way to implement this is by leveraging OpenResty and the ability to run custom logic at different stages of an HTTP request. In this specific case, before we send back the response to the client when he accesses `signup` endpoint, we could look at the response the database sent (and PostgREST forwarded) and if it was a success, we could run our custom function that sends the "welcome" email.

Info

It's important to remember that when in the Lua land running inside OpenResty/Nginx, you have complete freedom, you can do whatever you could have done in Node/PHP/Ruby, you can talk to other systems, run additional queries, make requests to other APIs.

If you only have a handful of these type of cases than doing it in OpenResty, before or after the request reaches the database, is the path you should choose. It's just less moving parts.

But there is a third way! This method allows for greater flexibility, decoupling and separation of concern. When you have to react to all kinds of database events and implement custom logic that talks to other systems or maybe execute some long running computation, you can use this pattern to decouple the main task from additional work that needs to happen. The trick here is to first recognize that the expensive computation you need to run (send email, generate report, talk to remote system) is a thing that can be executed asynchronously from the main request, you don't need to actually send the email to the user before you reply to the request with "signup success", you can send the email a fraction of a second later. If you look carefully, you will notice that this pattern comes up often, a lot of the tasks you need to perform do not have to necessarily be executed before you respond to the API request. This is exactly where a [Message queue](https://en.wikipedia.org/wiki/Message_queue) is useful. Whenever an event occurs in your system (a user just signed up for your service), you send a message to this queue and let a completely separate system (usually a simple script) take care or performing the additional tasks.

While this sounds complicated and fragile it's actually neither. For most of the systems, this pattern can be easily implemented using an `events` table. Your triggers will insert rows in this table whenever an even occurs triggered by an API request. Another script will read rows one by one from this table and do the additional work. Here is an [article](https://blog.2ndquadrant.com/what-is-select-skip-locked-for-in-postgresql-9-5/) explaining how to reliably do this using a table as a queue. You'll might hear that using the database as a queue is bad or does not scale, but this exact method [is used by Stripe](https://brandur.org/postgres-queues) and it works well for loads up to 100 events/s and most of the time that's all you need.

For simple things like sending emails the _table as a queue_ works great and does not require additional components and if you can get away with it, don't complicate things, use it.

But we all know that everyone wants live updates these days. It is next to impossible to do this with an `events` table. It would mean that every mutation in your database would have to be mirrored in this table. This is where the final piece of the puzzle comes in.

I am going to do a magic trick now and pull a [RabbitMQ](https://www.rabbitmq.com/) out of my hat. Just like the other components in our stack (OpenResty/PostgREST/PostgreSQL), it's an immensely powerful, flexible and battle tested piece of tech. It will give us yet another integration point and among other things will completely solve the "real-time updates" problem for us.

Before we get into that, we have not actually sent that "welcome" email yet. Let's do that.

Starter Kit already has everything in place to make sending messages from PostgreSQL to RebbitMQ easy. All we need to do now is to attach triggers to tables for which we want to monitor events/changes.

\-- db/src/data/tables.sql
\-- ...
create trigger send\_change\_event
after insert or update or delete on "user"
for each row execute procedure rabbitmq.on\_row\_change();

Let's look at what [that function](https://github.com/subzerocloud/subzero-starter-kit/blob/master/db/src/libs/rabbitmq/schema.sql) is doing

create or replace function rabbitmq.on\_row\_change() returns trigger as $$
  declare
    routing\_key text;
    row record;
  begin
    routing\_key :\= 'row\_change'
                   '.table-'::text || TG\_TABLE\_NAME::text || 
                   '.event-'::text || TG\_OP::text;
    if (TG\_OP \= 'DELETE') then
        row :\= old;
    elsif (TG\_OP \= 'UPDATE') then
        row :\= new;
    elsif (TG\_OP \= 'INSERT') then
        row :\= new;
    end if;
    perform rabbitmq.send\_message('events', routing\_key, row\_to\_json(row)::text);
    return null;
  end;
$$ stable language plpgsql;

So whenever something happens to a row, this function will take the row, turn it into JSON and generate a new message that will enter RabbitMQ. Before we look at where exactly that message ends up, we need to take a look at the `routing_key` since this is a very important part of what happens to the message when it is received by RMQ.

Info

If you are completely new to the concept of _message brokers_ and how they work, you can check out [RabbitMQ Tutorials](https://www.rabbitmq.com/getstarted.html)

When a new signup event occurs, this will generate an event that will go through the message broker and every interested party can react to it. The message will look something like

row\_change.table-user.event-INSERT.user-null
{
    "id":1,
    "firstname":"John",
    "lastname":"Smith",
    "email":"john.smith@gmail.com",
    "password":"encryptedpasshere",
    "user\_type":"webuser",
    "created\_on":"2017-02-20 08:56:44+00",
    "updated\_on":null
}

The routing key tells us exactly what happened (a row got inserted in the user table) and the message payload contains that actual row. This message will be sent on the channel `events`, which by default is linked to `amq.topic` exchange in RMQ. The message proxying between PostgreSQL and RabbitMQ is done by [pg-amqp-bridge](https://github.com/subzerocloud/pg-amqp-bridge) utility which is configured [here](https://github.com/subzerocloud/subzero-starter-kit/blob/master/docker-compose.yml#L90).

As you can imagine, all kind of events will flow through that exchange. When we write a script that wants to send emails on signup, it does not mean that this script will receive every event and have logic to ignore the ones that do not matter to this script. Your script (message consumer) can tell a message broker what type of messages it is interested in, and it will send only the ones that match your specification. In this particular case we are interested only when rows are added to the user table, so we `bind` to this exchange using the following routing key

row\_change.table-user.event-INSERT.#

If we wanted to monitor all events that happen on the user table, the binding key can be

#.table-user.#

or maybe we want to inspect all the events that were triggered by a specific user, in this case, the binding key would be

#.user-10

Info

It is important to understand that the format of the routing key is not something hard coded. You are free to change it to suit your needs. You are also free to send the events to any exchange you like, just remember to configure pg-amqp-bridge to forward them.

So at this stage, we have a system in place so that whenever someone signs up, this will trigger an event that will result in a message entering RabbitMQ. Let's see how we can actually send the email.

You can implement this script in any language you like but just to show how simple this can be, we'll do it using a shell script, just because we can. We could cheat and use something like [Mailgun](https://www.mailgun.com/) but instead, we'll actually send the email using our own Gmail account. Let's create our worker script

mkdir -p workers
touch workers/send\_signup\_email.sh
chmod +x workers/send\_signup\_email.sh

Here is the contens (rember to input your actual gmail credentials)

#!/bin/bash

\# requires the following command line executables to be installed
\# https://github.com/rmt/amqptools
\# http://caspian.dotconf.net/menu/Software/SendEmail/
\# https://stedolan.github.io/jq/

BASEDIR\=\`realpath $(dirname $(realpath $0))/../\`
source $BASEDIR/.env

GMAIL\_USER\=john.smith@gmail.com
GMAIL\_PASSWORD\=supperlongapplicationpassword
GMAIL\_FROM\="John Smith <john.smith@gmail.com>"

echo "starting signup event monitoring"
amqpspawn -h localhost -u $RABBITMQ\_DEFAULT\_USER -p $RABBITMQ\_DEFAULT\_PASS \\
    amq.topic "row\_change.table-user.event-INSERT.#" --foreground -q signups --durable | \\
    while read routing\_key file; do (
        \# start message processing logic
        NAME\=$(cat $file | jq -c -r '."name"')
        EMAIL\=$(cat $file | jq -c -r '."email"')
        rm $file

        printf "Sending signup email to ${NAME} <${EMAIL}\>... "

        sendEmail -o tls\=yes \\
        -xu $GMAIL\_USER \\
        -xp $GMAIL\_PASSWORD \\
        -s smtp.gmail.com:587 \\
        -f "$GMAIL\_FROM" \\
        -t "${NAME} <${EMAIL}\>" \\
        -u "Welcome to our service (Khumbu Icefall)" \\
        -m "We hope you enjoy your stay."
        \# end message processing logic
    ) done
echo "exiting ..."

Start up the script, it should look something like this:

$ ./workers/send\_signup\_email.sh 
starting signup event monitoring

Now let's make the signup call

curl \\
-H "Content-Type: application/json" \\
-H "Accept: application/vnd.pgrst.object+json" \\
-d '{"name":"John Smith","email":"john@smith.com","password":"pass"}' \\
http://localhost:8080/rest/rpc/signup

In the window where the worker is running, you'll see

...
Sending signup email to John Smith <john@smith.com> ...Jul 20 10:21:01 192-168-0-100 sendEmail\[33764\]: Email was sent successfully!
done

Did we just create a _microservice_ :)?

This script can run on completely separate hardware, decoupled from your main system. Another nice benefit is that even if the script is stopped or the hardware crashes because we specified `-q signups --durable`, RabbitMQ will hold on to all the signup events that happened in the meantime so when we restart the script, it will process the events retroactively.

subZero maintains a fork for (amqptools)\[[https://github.com/subzerocloud/amqptools](https://github.com/subzerocloud/amqptools)\] and provides (subzerocloud/amqptools)\[[https://hub.docker.com/r/subzerocloud/amqptools/](https://hub.docker.com/r/subzerocloud/amqptools/)\] docker image which you can use to simplify writing workers. With this image, all you need to do is write the (onmessage.sh)\[[https://github.com/subzerocloud/amqptools/blob/master/docker/onmessage.sh](https://github.com/subzerocloud/amqptools/blob/master/docker/onmessage.sh)\] script in order to implement the worker. You can add this worker to your `docker-config.yml` file like this.

signup\_email\_worker:
    image: subzerocloud/amqptools
    restart: unless-stopped
    volumes:
      - "./workers/send\_signup\_email.sh:/send\_signup\_email.sh"
    links:
      - rabbitmq
    environment:
      - AMQP\_HOST=rabbitmq
      # - AMQP\_PORT
      # - AMQP\_VHOST
      - AMQP\_USER=${RABBITMQ\_DEFAULT\_USER}
      - AMQP\_PASSWORD=${RABBITMQ\_DEFAULT\_PASS}
      # - EXCHANGE="amq.topic"
      - ROUTING\_KEY=row\_change.table-user.\*
      - AMQP\_QUEUE=create.customer
      - AMQP\_QUEUE\_DURABLE=true
      # - AMQP\_QUEUE\_PASSIVE
      # - AMQP\_QUEUE\_EXCLUSIVE
      # - AMQP\_QUEUE\_DURABLE
      - PROGRAM=/send\_signup\_email.sh
      # - MAX\_DELAY=30

When all else fails, take full control[¶](#when-all-else-fails-take-full-control "Permanent link")
--------------------------------------------------------------------------------------------------

subZero and this starter kit will take you 95% of the way when building your API. You will get there in no time and with an almost trivial amount of code. However, there are situations when it's hard to implement some use-cases within the constraints of a subZero based project. This is a general problem with systems that try to automate API creation, there are cases when it feels like you are fighting the tool and not solving your problem. You don't have to make that sacrifice here. It is possible to take full control of the flow.

An example when an automated API creation tool does not quite provide the flexibility needed is the case of integrating a payment processor. Usually, these types of systems provide their own SDKs that you have to use and they generate quite a lot of back and forth with a remote system. Some services embed in their core code integrations with specific well known 3rd party systems (slack, stripe, etc.) but you don't have the freedom to integrate with anything you want, you are limited to maybe triggering a few web hooks here and there.

Let's get back to our example. We've built our API (and frontend) and users are happy with it, it's time to get paid for our hard work. We'll charge our customers a one-time fee of $99 and we'll use Stripe for that. Our front end uses GraphQL (or REST) for communicating with the backend, however when integrating with Stripe, it is convenient to use it's recommended flow and that basically means you have to process a form submission.

Your frontend will display a page with a button like described in the [Stripe docs](https://stripe.com/docs/quickstart#checkout). Once the user goes through the process of entering his credit card, Stripe will hand over control to our system where based on a token, we create the customer and charge his credit card.

Now comes the interesting part, we hijack this URL and handle the request ourselves instead of letting it reach PostgREST and the database.

\-- openresty/nginx/conf/includes/http/server/locations/charge\_customer.conf
location /rest/rpc/charge\_customer {
    content\_by\_lua\_file '../lualib/user\_code/charge\_customer.lua';
}

Tip

I will be using Lua directly to implement this custom endpoint but you are free you route this URL to any stack you chose. You can use your traditional stack alongside PostgREST to handle a few tricky bits and to your api users it will be indistinguishable.

Info

To have these types of endpoint also apear in our GraphQL api, create a dummy stored procedures in your api schema like this

create or replace function charge\_customer() returns text as $$
    select 'dummy function'::text;
$$ stable language sql;

Here is a script (which could use some error handling) that will accomplish the task. Since we are taking over, we are responsible for checking the user is authenticated before making any additional work.

\-- openresty/lualib/user\_code/charge\_customer.lua
local jwt \= require 'resty.jwt'
local cjson \= require 'cjson'
local jwt\_secret \= os.getenv('JWT\_SECRET')
local stripe\_key \= 'sk\_test\_xxxxxxxxxxxxxxxx'
local stripe\_endpoint \= 'https://api.stripe.com/v1'
local stripe\_auth\_header \= 'Basic ' .. ngx.encode\_base64(stripe\_key ..':')

\-- check authorisation
local authorization \= ngx.var.http\_authorization
local token \= authorization:gsub('Bearer ', '')
local jwt\_obj \= jwt:verify(jwt\_secret, token)
if not (jwt\_obj and jwt\_obj.valid) then
    ngx.status \= ngx.HTTP\_UNAUTHORIZED
    ngx.exit(ngx.HTTP\_UNAUTHORIZED)
end
local user\_id \= jwt\_obj.payload.user\_id

\-- main script logic
ngx.req.read\_body()
local args \= ngx.req.get\_post\_args()
local stripe\_token \= args.stripeToken

\-- interact with stripe
\-- we could have used a nicer lua lib https://github.com/leafo/lua-payments but we can do this lo level also
local http \= require "resty.http"
local http\_client \= http.new()

\-- save the customer in stripe
local res \= http\_client:request\_uri(stripe\_endpoint .. '/customers', {
    method \= "POST",
    headers \= {
        \['Authorization'\] \= stripe\_auth\_header,
    },
    body \= ngx.encode\_args({
        \['metadata\[user\_id\]'\] \= user\_id,
        source \= stripe\_token
    })
})
local stripe\_customer\_id \= cjson.decode(res.body)\['id'\]

\-- charge the client
local res \= http\_client:request\_uri(stripe\_endpoint .. '/charges', {
    method \= "POST",
    headers \= {
        \['Authorization'\] \= stripe\_auth\_header,
    },
    body \= ngx.encode\_args({
        amount \= 9900,
        currency \= 'usd',
        description \= 'Khumbu Icefall Service',
        customer \= stripe\_customer\_id
    })
})

ngx.say('true')
\-- we could also redirect the user here

You don't have to fight subZero and jump through hoops to accomplish complicated flows, you can just sidestep it completely in a few specific cases and write your own custom code.
