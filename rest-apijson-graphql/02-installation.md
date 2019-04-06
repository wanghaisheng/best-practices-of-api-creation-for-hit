
Installation
============

### Docker[¶](#docker "Permanent link")

All subZero components are packaged as docker images, as such, the first thing to do is to install [Docker](https://www.docker.com/community-edition) for you platform.

### subzero-cli[¶](#subzero-cli "Permanent link")

To aid with the development process, we have build a set of command line tools to help you with various stages of your project.

Install `subzero-cli` as explained on it's [README page](https://github.com/subzerocloud/subzero-cli#install)

Bug

Due to a [bug](https://github.com/tj/commander.js/pull/604) in a upstream package, in windows you will need to replace git style subcommands with the full command name. Ex: `subzero base-project` should be executed as `subzero-base-project`

### Create a New Project[¶](#create-a-new-project "Permanent link")

subzero-cli provides you with a base-project command that lets you create a new project structure:

subzero base-project

? Enter the directory path where you want to create the project .
? Choose the starter kit (Use arrow keys)
  postgrest-starter-kit (REST) 
❯ subzero-starter-kit (REST & GraphQL) 

After the files have been created, you can bring up your application (API). In the root folder of application, run the docker-compose command

docker-compose up -d

The API server will become available at the following endpoints:

*   REST [http://localhost:8080/rest](http://localhost:8080/rest)
*   GraphiQL IDE [http://localhost:8080/graphiql](http://localhost:8080/graphiql)
*   GraphQL Simple Schema [http://localhost:8080/graphql/simple](http://localhost:8080/graphql/simple)
*   GraphQL Relay Schema [http://localhost:8080/graphql/relay](http://localhost:8080/graphql/relay)

Try a simple request

curl http://localhost:8080/rest/todos?select\=id,todo

### Debug/Development Dashboard[¶](#debugdevelopment-dashboard "Permanent link")

In the root folder of your project run the command

subzero dashboard

You should see a screen like this (From now on, we'll always have this running when iterating on our project) ![subzero-cli](../images/devtools.png "subzero-cli")
