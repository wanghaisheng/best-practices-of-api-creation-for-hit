
Architecture and Project Structure
==================================

### Architecture Choices[¶](#architecture-choices "Permanent link")

From the architecture point of view, this stack leverages the powerful features already present in each component instead of implementing everything in one monolithic component.

Why not leverage the capabilities of Nginx as an integration point and use it for routing requests based on custom runtime logic or have it handle caching or rate limiting?

Why not use PostgreSQL for more than a dumb data store, need to ask a complicated question of your data, create a view or a stored procedure. Need to send an email on signup, have a trigger add a message to a (RabbitMQ) queue to process it asynchronously. Need to define complicated rules for who can access what data, isn't this a core database feature? Why not use the database role system to describe those rules with simple semantics that are consistently enforced no matter the access path. It might sound complicated, but the reduction in boilerplate code is dramatic. The reduction of potential bugs is also very substantial, not only there is much less code for bugs to hide in, the remaining code is mostly declarative in nature. You'll be defining your APIs and not writing them, subtle but important difference.

### Project Structure[¶](#project-structure "Permanent link")

Let's now look at the directory layout of a project

.
├── db                        \# Database schema source files and tests
│   └── src                   \# Schema definition
│       ├── api               \# Api entities avaiable as REST endpoints
│       ├── data              \# Definition of source tables that hold the data
│       ├── libs              \# A collection modules of used throughout the code
│       ├── authorization     \# Application level roles and their privileges
│       ├── sample\_data       \# A few sample rows
│       └── init.sql          \# Schema definition entry point
├── openresty                 \# Reverse proxy configurations and Lua code
│   ├── lualib
│   │   └── user\_code         \# Application Lua code
│   ├── nginx                 \# Nginx files
│   │   ├── conf              \# Configuration files
│   │   └── html              \# Static frontend files
│   ├── Dockerfile            \# Dockerfile definition for production
│   └── entrypoint.sh         \# Custom entrypoint
├── tests                     \# Tests for all the components
│   ├── db                    \# pgTap tests for the db
│   ├── graphql               \# GraphQL interface tests
│   └── rest                  \# REST interface tests
├── docker-compose.yml        \# Defines Docker services, networks and volumes
└── .env                      \# Project configurations

The first thing to look at is the docker-compose.yml file, it's the place where all components in the stack and relations between them are defined. You'll notice that each component also has a top level directory where you'll find everything related to the configuration of that component and the code that lives within it.

#### OpenResty[¶](#openresty "Permanent link")

There are a few interesting directories here, `nginx/html` has the most obvious purpose, that is where you place your static assets for your frontend. Then you have `lualib/user_code`. This is where you'll place your Lua code that will run in the proxy. This code will be very lightweight, simple functions that implement some actions before or after the request goes to the database. `nginx/conf` is the more "heavy" of the bunch. If you look at it, you'll see a lot of files in a nested structure. Don't worry, you won't have to touch them, they are already setup to be enough for 90% of applications, but whenever you need something custom, the full configuration is there. All of the nginx configuration could have been implemented in a single file, but we've chosen to break it up into separate files to allow for easy merging of changes from the upstream (this project). When you want to add custom configurations, you add your file (and not change an existing one), and this eliminates merge conflicts. If you are curious, you can take a more careful look at the config, but you don't need to in order to be productive.

#### PostgREST[¶](#postgrest "Permanent link")

Besides the configuration which is done through env vars in the docker-config.yml file, there is not much PostgREST needs. That's why it does not have dedicated folder.

#### PostgreSQL (db)[¶](#postgresql-db "Permanent link")

The database is where the bulk of your work will be. The SQL (or any other supported language) will be executed using `psql`; thus we have at our disposal meta commands. Specifically, we leverage the `\ir` (include relative) to break up our code into multiple files. The entrypoint is the `init.sql` file. It includes all other (top level) modules. The `init.sh` can be used to add custom configuration to (development) PostgreSQL instance at boot time. We use it to configure detailed query logging.

Since the entrypoint is one script and `\ir` commands drive the order in which code is included and executed, the folder structure here is entirely up to you. We think that this structure works quite well. First you have your `data` directory, and usually, you have a corresponding schema in the database to match the name. This is where you will place the definitions of your tables and indexes. This schema is the sensitive area where you need to be careful when doing migrations (we'll look at it later) that is why we think it's better to separate these definitions in their own directory and schema. Every other schema will have just logic comprised of views, stored procedures, roles and their privileges, code that can be safely deleted and recreated (almost like uploading a new version of the files) without the risk of damaging the data. Then there is the `api` directory (with the matching schema). This is the schema that we'll be exposed as HTTP endpoints. Every entity in this schema will be available as an HTTP endpoint. Although it's possible to define here tables directly, you want to decouple your internal models from the API. That's why we have the `data` schema where we store our tables and then we have views in the `api` schema that draw data from those tables, even though most of them are "mirrors" of those tables. Throughout your view/stored procedures definitions, you'll find cases where you can extract particular logic in a separate function. Those functions belong in the `libs` directory. You'll also want to namespace them using schemas.
