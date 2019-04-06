

Overview
========

What We're Building[¶](#what-were-building "Permanent link")
------------------------------------------------------------

As tradition goes we'll build and API for a To-do app but we'll go much further than the concept of tasks. We'll have users, clients, projects, tasks and (real-time) comments, all the core models to build a complete API for a project management tool. We'll also have a codename for this secret project, "Khumbu Icefall", you get extra points if you can figure out why I chose the name.

Unlike most tutorials that just explain a few basic concept with a toy example, this one will take you through all the steps of building a real-world, complex API that is ready to go into production. You've probably heard (marketing) phrases like "build a production ready API in 5 minutes", tried it and found out it's ... marketing, a production ready toy, not a real system. This tutorial is not going to take 5 minutes but it's also not going to take days to complete (or even months, because usually, that is the timeframe you are looking at when building APIs). We are looking at 1-2 hours of your time. If you are in a hurry or maybe you'd like to see why you should spend two hours on this, skip to the end and try interacting with a live version of Khumbu Icefall (KI) or run it directly on your computer.

When presented with systems like subZero, in some cases the conversation goes like "Yes, I guess it's an excellent tool for prototyping but my API is not a 1:1 map of my data models and it has a lot of business logic so I can not use it in production". In this tutorial, we'll show how a subZero based API is not coupled to your tables and how you can implement your custom business logic and even integrate/interface with 3rd party systems (and not just by calling webhooks). For most of the tutorial we'll be using the REST interface to interact with the system because it's easier to copy/paste curl lines, but while you are growing your REST API, GraphQL is right next to it.

Installation[¶](#installation "Permanent link")
-----------------------------------------------

Follow the [installation instructions](../installation/) and familiarize yourself a bit with the [project structure](../architecture-and-project-structure)

Edit the file `.env` file and give the project a distinct name

COMPOSE\_PROJECT\_NAME\=khumbuicefall

Bring up the system to check everything works

docker-compose up -d \# wait for 5-10s before running the next command
curl http://localhost:8080/rest/todos?select\=id

The result should look something like this

\[{"id":1},{"id":3},{"id":6}\]

Bring up the [subzero dashboard](../Installation#subzero-cli)

subzero dashboard

Now that we have the API and `subzero dashboard` up and running, let's see what we can do.

Run this request again and see it reflected on the PostgreSQL logs tab in the subzero dashboard

curl http://localhost:8080/rest/todos?select\=id

Now let's make an authorized request

export JWT\_TOKEN\=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoxLCJyb2xlIjoid2VidXNlciJ9.uSsS2cukBlM6QXe4Y0H90fsdkJSGcle9b7p\_kMV1Ymk
curl -H "Authorization: Bearer $JWT\_TOKEN" http://localhost:8080/rest/todos?select\=id,todo

\[{"id":1,"todo":"item\_1"},{"id":2,"todo":"item\_2"},{"id":3,"todo":"item\_3"},{"id":6,"todo":"item\_6"}\]

Notice the first item in the list `{"id":1,"todo":"item_1"}`. Open the file `db/src/sample_data/data.sql` and change `item_1` to `updated` and save the file.

Run the last request again

curl -H "Authorization: Bearer $JWT\_TOKEN" http://localhost:8080/rest/todos?select\=id,todo

\[{"id":1,"todo":"updated"},{"id":2,"todo":"item\_2"},{"id":3,"todo":"item\_3"},{"id":6,"todo":"item\_6"}\]

This process works for any `*.sql` `*.lua` `*.conf` file, you just make the change, save the file and you can run your next request, assuming your changes were valid (when working with sql, look at the PostgreSQL window).
