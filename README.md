
## Hit the ground running
GraphQL & APIJSON & REST API for your database

一站式工具包——零成本解决应用开发、系统集成过程中API构建的问题

Stop spending months developing backend APIs for your apps.
With subZero, you’ll be ready in a few days — and you can get back to building your product.

### Move Fast

Imagine being able to have your API 80% ready after a few quick steps. Define your tables and views, set constraints on your columns, grant privileges to users, and you’re there — all while using standard SQL statements. To finish the last 20%, you can use real languages to implement your custom logic, not just SDKs and webhooks.

### Stay Flexible
By adopting GraphQL as the language for communication with your API, you remove any coupling between your frontend code and the backend. Combined with the ability to choose any database schema you like, you get a no lock in guarantee.

### Own Your Data

Unlike other solutions, we never require you to store your data in a proprietary system. The data lives in your database, in the format you choose — just like it’s supposed to. If you already have your database and data, you don’t have to import it anywhere or change the structure. subZero will take what you have and work with it, without any extra steps.


###  Unparalleled development speed
![](https://subzero.cloud/img/clock.jpg)
We’ve gone through the process of building APIs and we know how long it takes to get everything set up. You always think you’ll be able to get it up quickly, and then two months later you’re still stuck on your API before you’ve even been able to start on your product.
subZero cuts that development time down drastically.

### Lightning fast
On a system composed of an AWS t2.micro instance running subZero, connected to a RDS db.t2.micro database, we were seeing more than 500 requests per second, with latencies in the range of 5 ms — 20 ms on a three table join.
To put that in perspective, you get 1,296,000,000 requests/month on $21 hardware, which is 100 times more than our competitors give you for the same price, even after factoring in the licensing costs of our software.


### various database supported 

MySQL, PostgreSQL,Oracle,MS SQL , SQLite, MongoDB, CouchDB, and others.
Dgraph,Elasticsearch, Neo4j, 

### GraphQL, a query language for your API

GraphQL is a query language for APIs which provides a complete and understandable description of the data in your API. The upsides are many:

It gives clients the power to ask for exactly what they need
It makes it easier to evolve APIs over time
It enables powerful developer tools
Apps that use GraphQL don’t have to sacrifice speed for stability (or vice versa) because they control the data they get, not the server. While typical REST APIs require loading from multiple URLs, GraphQL APIs get all the data your app needs in a single request. This mean your app will always be lightning fast — even on slow mobile network connections.


### APIJSON


### FHIR

### A few more reasons to try this fullstack:
Flexible GraphQL & REST API out of the box
Extremely short time to production (we’re talking days)
Efficient queries — no N+1 problem
Built-in scriptable caching system
Easy deployment as Docker containers
Use the full power of SQL/Lua/Python/Javascript
Support for GraphQL/Relay specifications
Real-time updates using a Messaging Server




### reference

The following is a list of tools that automatically expose a REST, GraphQL, or another kind of API for your database, as well as databases with a built-in HTTP API.

<table>
  <tr>
    <th>Project name/link</th>
    <th>Database(s) supported</th>
    <th>API type</th>
    <th>Implementation language</th>
    <th>License</th>
    <th>GitHub stats</th>
    <th>Notes</th>
  </tr>
  <tr>
    <td><a href="https://github.com/arangodb/arangodb">ArangoDB</a></td>
    <td>ArangoDB</td>
    <td>REST</td>
    <td>C++</td>
    <td>Apache 2.0</td>
    <td>7040 ★; 44129 commits, latest 2019-01-14</td>
    <td>A database with a built-in REST API. <a href="https://hub.docker.com/r/arangodb/arangodb/">Official Docker image</a>.</td>
  </tr>
  <tr>
    <td><a href="https://github.com/apache/couchdb">CouchDB</a></td>
    <td>CouchDB</td>
    <td>REST</td>
    <td>Erlang</td>
    <td>Apache 2.0</td>
    <td>4010 ★; 11526 commits, latest 2019-01-14</td>
    <td>A database with a built-in REST API. <a href="https://hub.docker.com/r/_/couchdb/">Official Docker image</a>.</td>
  </tr>
  <tr>
    <td><a href="https://github.com/simonw/datasette">Datasette</a></td>
    <td>SQLite 3</td>
    <td>REST</td>
    <td>Python 3</td>
    <td>Apache 2.0</td>
    <td>2164 ★; 592 commits, latest 2019-01-13</td>
    <td>Read-only. <a href="https://hub.docker.com/r/terranodo/datasette/">Official Docker image</a>.</td>
  </tr>
  <tr>
    <td><a href="https://github.com/dgraph-io/dgraph">Dgraph</a></td>
    <td>Dgraph</td>
    <td><a href="https://docs.dgraph.io/query-language/">GraphQL+-</a>, a GraphQL derivative</td>
    <td>Go</td>
    <td>Apache 2.0</td>
    <td>7323 ★; 2842 commits, latest 2019-01-15</td>
    <td>A database with a built-in GraphQL-like API. <a href="https://hub.docker.com/r/dgraph/dgraph/">Official Docker image</a>.</td>
  </tr>
  <tr>
    <td><a href="https://github.com/dreamfactorysoftware/dreamfactory">DreamFactory</a></td>
    <td>MySQL, PostgreSQL, SQLite, MongoDB, CouchDB, and <a href="https://www.dreamfactory.com/products">others</a>.</td>
    <td>REST</td>
    <td>PHP 5</td>
    <td>Apache 2.0, proprietary (optional extras)</td>
    <td>892 ★; 813 commits, latest 2018-09-19</td>
    <td><a href="https://hub.docker.com/r/dreamfactorysoftware/df-docker/">Official Docker image</a>.</td>
  </tr>
  <tr>
    <td><a href="https://github.com/pyeve/eve">Eve</a></td>
    <td>MongoDB; extensions for Elasticsearch, Neo4j, SQLAlchemy (SQL databases).</td>
    <td>REST</td>
    <td>Python 2/3</td>
    <td>BSD (three-clause)</td>
    <td>5423 ★; 2910 commits, latest 2018-10-11</td>
    <td>The SQLAlchemy extension isn't automatic. It requires the user to write SQLAlchemy mappings.</td>
  </tr>
  <tr>
    <td><a href="https://github.com/hasura/graphql-engine">Hasura GraphQL Engine</a></td>
    <td>PostgreSQL</td>
    <td>GraphQL</td>
    <td>Haskell</td>
    <td>GNU AGPLv3</td>
    <td>5959 ★; 560 commits, latest 2019-01-15</td>
    <td><a href="https://hub.docker.com/r/hasura/graphql-engine/">Official Docker image</a>.</td>
  </tr>
  <tr>
    <td><a href="https://bitbucket.org/prometheus/htsql/src">HTSQL</a></td>
    <td>MySQL, PostgreSQL, SQLite (free); Oracle, MS SQL (proprietary)</td>
    <td>REST</td>
    <td>Python 2</td>
    <td>GNU AGPLv3, proprietary (Oracle and MS SQL support)</td>
    <td>n/a</td>
    <td></td>
  </tr>
  <tr>
    <td><a href="https://github.com/neo4j-graphql/neo4j-graphql">neo4j-graphql</a></td>
    <td>Neo4j</td>
    <td>GraphQL</td>
    <td>Kotlin</td>
    <td>Apache 2.0</td>
    <td>279 ★; 142 commits, latest 2019-01-09</td>
    <td>Can generate a GraphQL API from an existing database or derive a new database model from a GraphQL schema and auto-generate the resolvers.</td>
  </tr>
  <tr>
    <td><a href="https://github.com/orientechnologies/orientdb">OrientDB</a></td>
    <td>OrientDB</td>
    <td>REST</td>
    <td>Java</td>
    <td>Apache 2.0</td>
    <td>3730 ★; 17650 commits, latest 2019-01-14</td>
    <td>A database with a built-in REST API. <a href="https://store.docker.com/images/orientdb">Official Docker image</a>.</td>
  </tr>
  <tr>
    <td><a href="https://github.com/mevdschee/php-crud-api">PHP-CRUD-API</a></td>
    <td>MySQL, PostgreSQL, MS SQL Server.</td>
    <td>REST</td>
    <td>PHP 7</td>
    <td>MIT</td>
    <td>1836 ★; 1282 commits, latest 2018-12-11</td>
    <td>Supports GIS + automatic OpenAPI 3.0 docs.</td>
  </tr>
  <tr>
    <td><a href="https://github.com/graphile/postgraphile">PostGraphile</a></td>
    <td>PostgreSQL</td>
    <td>GraphQL</td>
    <td>TypeScript (Node.js)</td>
    <td>MIT</td>
    <td>6565 ★; 1025 commits, latest 2019-01-11</td>
    <td>Formerly &quot;PostGraphQL&quot;, <a href="https://hub.docker.com/r/postgraphql/postgraphql/">Official Docker image</a>.</td>
  </tr>
  <tr>
    <td><a href="https://github.com/PostgREST/postgrest">PostgREST</a></td>
    <td>PostgreSQL</td>
    <td>REST</td>
    <td>Haskell</td>
    <td>MIT</td>
    <td>11722 ★; 1441 commits, latest 2018-12-12</td>
    <td><a href="https://hub.docker.com/r/postgrest/postgrest/">Official Docker image</a>.</td>
  </tr>
  <tr>
    <td><a href="https://github.com/prest/prest">pREST</a></td>
    <td>PostgreSQL</td>
    <td>REST</td>
    <td>Go</td>
    <td>MIT</td>
    <td>1916 ★; 440 commits, latest 2018-09-04</td>
    <td><a href="https://hub.docker.com/r/prest/prest/">Official Docker image</a>.</td>
  </tr>
  <tr>
    <td><a href="https://github.com/SoftInstigate/restheart">RESTHeart</a></td>
    <td>MongoDB</td>
    <td>REST</td>
    <td>Java</td>
    <td>GNU AGPLv3</td>
    <td>512 ★; 1596 commits, latest 2019-01-03</td>
    <td><a href="https://hub.docker.com/r/softinstigate/restheart/">Official Docker image</a>.</td>
  </tr>
  <tr>
    <td><a href="https://github.com/jeffknupp/sandman2">sandman2</a></td>
    <td>All supported by SQLAlchemy (MySQL, PostgreSQL, SQLite, Oracle, MS SQL, and others).</td>
    <td>REST</td>
    <td>Python 2/3</td>
    <td>Apache 2.0</td>
    <td>876 ★; 159 commits, latest 2019-01-10</td>
    <td><a href="https://hub.docker.com/r/jeffknupp/sandman2/">Official Docker image</a>.</td>
  </tr>
  <tr>
    <td><a href="https://subzero.cloud">subZero</a></td>
    <td>PostgreSQL</td>
    <td>REST and GraphQL</td>
    <td>Haskell, Lua</td>
    <td>Proprietary</td>
    <td>n/a</td>
    <td></td>
  </tr>
  <tr>
    <td><a href="https://github.com/bradleyboy/tuql">tuql</a></td>
    <td>SQLite 3 or SQL infile</td>
    <td>GraphQL</td>
    <td>JavaScript (Node.js)</td>
    <td>MIT</td>
    <td>346 ★; 52 commits, latest 2018-02-16</td>
    <td></td>
  </tr>
  <tr>
    <td><a href="https://github.com/nicolasff/webdis">Webdis</a></td>
    <td>Redis</td>
    <td>REST</td>
    <td>C</td>
    <td>BSD (two-clause)</td>
    <td>2081 ★; 399 commits, latest 2018-08-11</td>
    <td>Supports pub/sub with chunked transfer encoding and WebSockets.</td>
  </tr>
  <tr>
    <td><a href="https://github.com/o1lab/xmysql">xmysql</a></td>
    <td>MySQL</td>
    <td>REST</td>
    <td>JavaScript (Node.js)</td>
    <td>MIT</td>
    <td>3431 ★; 266 commits, latest 2018-09-28</td>
    <td><a href="https://hub.docker.com/r/markuman/xmysql/">Official Docker image</a>.</td>
  </tr>
  <tr>
    <td><a href="https://github.com/BjoernKW/ZenQuery">ZenQuery</a></td>
    <td>PostgreSQL, MySQL, IBM Db2, Oracle Database, Microsoft SQL Server and <a href="https://github.com/BjoernKW/ZenQuery#database">others</a></td>
    <td>REST</td>
    <td>Java (JavaScript for the front-end)</td>
    <td>Apache 2.0</td>
    <td>37 ★; 283 commits, latest 2018-10-16</td>
    <td>Read-only.</td>
  </tr>
</table>