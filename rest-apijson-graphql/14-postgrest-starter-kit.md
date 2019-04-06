
PostgREST Starter Kit
=====================

### But Why?[¶](#but-why "Permanent link")

So what is this? Isn't PostgREST supposed to be just a binary that you connect to your database and you instantly get a REST API? Why the starter kit? Yes, that is all true, and for simple use cases, particularly in cases where you want to expose some data for reading, but if you want to replace your backend stack completely, then things are a little more complicated.

Just after the immediate excitement for all the great things PostgREST brings, you are faced with a list of questions that you are not sure how to solve (and you did know the answer for them in your old MVC stack). For this reason, many people just assume that PostgREST is only useful for prototyping, I assure you it's not just for that, it's a solid candidate for production.

### Features[¶](#features "Permanent link")

Here is the list of things this starter kit will solve for you if you decide to use PostgREST

*   Easy to set up dev environment using Docker
*   Standardized directory structure
*   Debugging and live code reloading
*   Authentication/authorization flow that is easy to adapt
*   Unit and integration tests infrastructure
*   Reverse proxy configuration with predefined hooks in Lua where you can add custom logic at every step of the HTTP request
*   Scripts and tools to setup a CI/CD pipeline

Any project that aims to go for production will eventually need all these things (and I bet sooner than later), so it's better to just start with a setup that has this. This starter kit, however, is not the way to learn PostgREST and how it works, it's a structure to help you set up the entire backend for your application.

Since this starter kit packs a lot of functionality, it's obvious that it's a little bit more complicated than a promise of a single binary, but the upside is that this can completely cover your backend needs. The biggest productivity boost this setup gives you is the ability to work with code that lives in the database (tables/views/stored procedures/rules) just as you work with scripts in other interpreted languages, you just save a file, and then you can just check the result.
