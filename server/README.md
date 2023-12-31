# OIDC Take Home Exercise

This is a NodeJS/Express app that acts as an OpenID Connect Identity Provider
Note: this application only implements a minimal subset of the OpenID Connect Standard

## Running and Building

### Docker

Build application images/network and run application stack on network containers, publishing to port 8080:

```bash
$ docker compose up --build
```

The application is fully built and running when you see an info-level log with the message `Server listening on port 8080!`
      - Once the server is up and running, you can view the API's Documentation at localhost:8080/api-docs/
      - Please note that the application may take some time to compile the underlying images on the first build

### Local Node

Use the following when developing / making changes to the application

In the root project directory, execute the following bash commands (after starting postgres server with configuration matching connection specified in .env)

1. Install node modules: `npm install`
2. Generate openapi.yaml api schema (only necessary on changes to controllers) `npm run gen:tsoa`
3. Generate/migrate database schema, generate Prisma client (including updating types): `npm run prisma:migrate`
   - If no changes to Prisma schema, command will note no changes applied
   - If changes to Prisma schema, command will prompt for name of new migration (for corresponding changes to Prisma schema)
4. Build the /dist directory: `npm run build`
3. Start the server: `npm run start`


#### Useful Commands

- Reset DB, Generate/Migrate DB Schema and gen Prisma client: `npm run prisma:migrate-reset`

## Potential Limitations & Edge Cases

### Limitations
1. No endpoint for deactivation or deletion of tokens.
2. No token expiration / refresh functionality.
3. `authorize` endpoint assumes fixed values for `response_type`, `response_mode`, `code_challenge_method`, and `scope`
      - does not account for other options and will handle request in the same way regardless of the values sent
4. Could be concerns around horizontal scalability that comes with the relational database, as well as the poor time complexity of SQL joins that may become apparent with increased database complexity.
5. With no login page integration, this server has the limitation of lacking the main functionality of a OpenId Connect Provider that is authentication - determining who is attempting to access resources.

### Edge Cases
1. Right now, it would be possible for any malicious client on a public network to access the endpoint and receive an authorization token (lacking client authentication) - no clientID or client secret.
2. JWT is currently hashed and stored in database, could present security issues if database is compromised, but hashing strategy should mitigate this risk.
3. Handling session timeouts: currently, no timeout/expiration on the initial authorization code that is granted to the user.
4. Only handling parameters as expected from exercise documentation - validation error will occur if server is passed extra parameters such as clientID, client Secret, state, etc.

## NOTES ON TOOLS/LIBRARIES

- Winston: https://github.com/winstonjs/winston
  - Logging library
- TSOA: https://tsoa-community.github.io/docs/
  - Library used for generating API spec and routes from TypeScript _controllers_ and _models_
  - Library used for custom middlware (authorization)
    - https://github.com/lukeautry/tsoa/pull/1123
- Prisma: https://www.prisma.io/docs/
  - Database management tool
    - Used for _migrations_ and _ORM_ functionality

## NOTES ON PROJECT STRUCTURE/DIRECTORIES

### SRC FOLDER : `/back/src/`

- The `src` directory is where we do most of our development: it contains the typescript assets that will eventually be transpiled into JavaScript and stored in `/back/dist`
- Consists of subdirectories that constitute our backend functionality
  - _config_
    - Contains configuration for our express server (_app.ts_), along with configuration for logger:
      - _winston.config_
  - _controllers_
    - Contains controllers to handle requests and be used for route/API spec generation
  - _models_
    - Contains our model (_DTO_) class declarations
  - _service_
    - Contains API's business logic --> accessed between the controller and repository layers
  - _repository_
    - Repository layer: contains functions to interact with our database through the use of Prisma client
  - _routes_
    - Generated by tsoa through our controllers --> exports _RegisterRoutes_ middleware which is used in `/src/config/app.ts` to give direct our requests to their associated controllers
  - _utils_
    - Contains utility functions/classes to be used in services

### API : `/openapi.yaml`

- This file is generated by our backend code (using _TSOA_): schemas (_objects/data structures_) are generated from our models and paths (_routes/endpoints_) are generated from our controllers
  - **EXAMPLE**
    - Code in _jwks.controller.ts_ generates any `/jwks` paths in _openapi.yaml_ --> also generates the schema for any models referenced in controller 
  - The _openapi.yaml_ schema adheres to the _Swagger-OpenAPI 3.0_ specification
    - This specification allows us to describe our entire API in one file, including the _following attributes_:
      - Available endpoints (/users) and operations on each endpoint (GET /users, POST /users)
      - Operation parameters input and output for each operation
      - Authentication methods
      - Contact information, license, terms of use and other information


### PRISMA FOLDER : `/back/prisma/`

- _PRISMA_ is an open source ORM which allows us to query and manipulate data from our database using object-oriented programming.
  - The Prisma folder contains all assets to build our data model and generate an ORM client to interact with our database. Below is an overview the different assets in our Prisma folder:
    - `Migrations` _(directory)_: this directory consists of migration directories that hold SQL scripts representing the history of changes that we have made to our data model throughout development
      - This directory is the source of truth for the current state and history of our data model
      - Allows us to keep our database schema in sync with our Prisma schema, and maintain existing data in our database
    - `Schema.prisma` _(file)_: This is the Prisma schema file: this file is where we define our application models in Prisma’s data modeling language. It also contains the connections to our database and defines a generator (or multiple).
      - The main generator we are using is _prisma-client-js_: this generator reads our Prisma schema and generates the Prisma client code
      - We import the Prisma client into our services and use it to interact with our database (_using prisma client in the same way we would use a traditional ORM tool_)

### LOG FOLDER : `/back/logs/`

- The log folder consists of log files containing application’s logging history
  - Through use of _Winston_ (logging library), we can configure multiple loggers for use in different levels of our application
    - With this configuration, we are able to how / where we save the history of logs for different loggers
    - For example, we configure _servicesLogger_ to both log to the _console_ as well as our specified _services.log_ file – this allows us to both _observe live logs_ from the system as well as _maintain a history of these logs for future evaluation_