

Motivation
==========

subZero is a software development stack, with the primary focus of building GraphQL and REST APIs backed by a PostgreSQL database. The stack establishes a single declarative source of truth: the data itself. The structural constraints and permissions in the database determine the API endpoints and operations. Using subZero is an alternative to manual CRUD programming.

### Move Fast[¶](#move-fast "Permanent link")

Imagine being able to have your API 80% ready after a few quick steps. Define your tables and views, set constraints on your columns, grant privileges to users, and you’re there — all while using standard SQL statements. To finish the last 20%, you can use real languages to implement your custom logic, not just SDKs and webhooks.

### Stay Flexible[¶](#stay-flexible "Permanent link")

By adopting GraphQL or REST as the protocol for communication with your API, you remove any hard coupling between your frontend code and the backend. Instead of having your database structure conform to some format of a framework, subZero let's your choose whatever structure you need for the problem you are actually solving. The data lives in your database, in the format you choose. If you already have your database and data, you don’t have to import it anywhere or change the structure. subZero will take what you have and work with it.

### Unix Philosophy[¶](#unix-philosophy "Permanent link")

Each component in the subZero stack has a focused scope. Rather then building a big ball of mud, use a collection of sharp tools that do one thing well and can be composed to implement a solution. Instead of reinventing the wheel, we rely on well established tools with decades of research and development and use them to the full extent of their capabilities.
