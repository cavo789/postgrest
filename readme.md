<!-- This file has been generated by the concat.sh script. -->
<!-- Don't modify this file manually (you'll loose your changes) -->
<!-- but run the tool once more -->
<!-- Last refresh date: Tuesday, January 10, 2023, 09:34:18 -->

# PostgREST tutorial

**Personnal learning** How to use [https://postgrest.org/en/stable/tutorials/tut0.html](https://postgrest.org/en/stable/tutorials/tut0.html)

![Banner](./banner.svg)

> A docker-compose example with Postgres, PostREST and Swagger UI: [https://github.com/mattddowney/compose-postgrest](https://github.com/mattddowney/compose-postgrest)

<!-- table-of-contents - start -->
* [Introduction](#introduction)
* [Set-up](#set-up)
  * [Set-up](#set-up)
    * [Postgres](#postgres)
    * [PostgREST](#postgrest)
* [Set-up a sample database for this tutorial](#set-up-a-sample-database-for-this-tutorial)
  * [Create the tutorial.conf file](#create-the-tutorial-conf-file)
* [Using PostgREST](#using-postgrest)
  * [Read data from the database](#read-data-from-the-database)
    * [Making some calls](#making-some-calls)
      * [Call a stored procedure](#call-a-stored-procedure)
      * [Define the output format](#define-the-output-format)
      * [Don't return an array](#don-t-return-an-array)
      * [Joins](#joins)
  * [Interact with the database with patch, post, put or delete](#interact-with-the-database-with-patch-post-put-or-delete)
    * [Add a trusted user](#add-a-trusted-user)
    * [Make a secret](#make-a-secret)
  * [Allow "tr" to process non-utf8 byte sequences](#allow-tr-to-process-non-utf8-byte-sequences)
  * [read random bytes and keep only alphanumerics](#read-random-bytes-and-keep-only-alphanumerics)
  * [PASSWORD MUST BE AT LEAST 32 CHARS LONG](#password-must-be-at-least-32-chars-long)
  * [add this line to tutorial.conf:](#add-this-line-to-tutorial-conf)
    * [Sign a token](#sign-a-token)
    * [Make a request](#make-a-request)
* [Laravel](#laravel)
* [Tips](#tips)
  * [URL rewriting](#url-rewriting)
  * [support /endpoint/:id url style](#support-endpoint-id-url-style)
* [Hardening PostgREST](#hardening-postgrest)
* [Monitoring](#monitoring)
* [Implement PostgREST in an existing Docker project](#implement-postgrest-in-an-existing-docker-project)
  * [Add services in docker-compose.yml](#add-services-in-docker-compose-yml)
  * [Create user and give permissions in your DB_DATABASE](#create-user-and-give-permissions-in-your-db-database)
* [Links](#links)
  * [French tutorials](#french-tutorials)<!-- table-of-contents - end -->

## Introduction

> PostgREST is a standalone web server that turns your PostgreSQL database directly into a RESTful API. The structural constraints and permissions in the database determine the API endpoints and operations.
>
> [PostgREST](https://postgrest.org/en/stable/)

PostgREST is compliant with [OpenAPI](https://swagger.io/specification/). It's then possible to auto-document his routes using the `Swagger UI` Docker image.

## Set-up

### Set-up

#### Postgres

Get and run a postgres database server: 

```bash
docker run --name tutorial -p 5433:5432 \
    -e POSTGRES_PASSWORD=mysecretpassword \
    -d postgres
```

#### PostgREST

Officiel Docker image: [https://hub.docker.com/r/postgrest/postgrest](https://hub.docker.com/r/postgrest/postgrest). To configure the image, we need to set environment variables: [https://postgrest.org/en/stable/configuration.html#env-variables-config](https://postgrest.org/en/stable/configuration.html#env-variables-config)

Example of a `docker-compose.yml` with both the Postgres and PostgREST services: [https://postgrest.org/en/stable/install.html#containerized-postgrest-and-db-with-docker-compose](https://postgrest.org/en/stable/install.html#containerized-postgrest-and-db-with-docker-compose)

If you want to have a visual overview of your API in your browser you can add swagger-ui to your docker-compose.yml:

```yaml
swagger:
  image: swaggerapi/swagger-ui
  ports:
    - "8080:8080"
  expose:
    - "8080"
  environment:
    API_URL: http://localhost:3000/
```

If you need it manually, here is how to proceed:

* Download the binary from https://github.com/PostgREST/postgrest/releases/download/v10.1.1/postgrest-v10.1.1-linux-static-x64.tar.xz
* Run `tar xJf postgrest-v10.1.1-linux-static-x64.tar.xz`

## Set-up a sample database for this tutorial

Run a SQL console:

```bash
docker exec -it tutorial psql -U postgres
```

Execute:

```sql
create schema api;

create table api.todos (
  id serial primary key,
  done boolean not null default false,
  task text not null,
  due timestamptz
);

insert into api.todos (task) values
  ('finish tutorial 0'), ('pat self on back');

create role web_anon nologin;

grant usage on schema api to web_anon;
grant select on api.todos to web_anon;

create role authenticator noinherit login password 'mysecretpassword';
grant web_anon to authenticator;
```

`\q` pour quitter

### Create the tutorial.conf file

Create `tutorial.conf` with tis content:

```text
db-uri = "postgres://authenticator:mysecretpassword@localhost:5433/postgres"
db-schemas = "api"
db-anon-role = "web_anon"
```

And run the postgrest server:

```bash
./postgrest tutorial.conf
```

More doc about the configuration file can be retrieved on [https://postgrest.org/en/stable/configuration.html](https://postgrest.org/en/stable/configuration.html)

## Using PostgREST

### Read data from the database

From now, we can try it:

```bash
curl http://localhost:3000/todos | jq
```

(if `jq` isn't installed yet, please run `sudo apt install jq`)

That call will return the list of all todos defined in the `todos` table as a JSON string.

#### Making some calls

* Get the entire table: `curl http://localhost:3000/todos | jq`

    ```json
    [
        {
            "id": 1,
            "done": false,
            "task": "finish tutorial 0",
            "due": null
        },
        {
            "id": 2,
            "done": false,
            "task": "pat self on back",
            "due": null
        }
    ]
    ```

* Only a given record (`id=2`): `curl http://localhost:3000/todos\?id\=eq.2 | jq`

    ```json
    [   
        {
            "id": 2,
            "done": false,
            "task": "pat self on back",
            "due": null
        }
    ]
    ```

* Full text search on a field: `curl "http://localhost:3000/todos?task=fts.tutorial" | jq` ([doc](https://postgrest.org/en/stable/api.html#full-text-search))

    ```json
    [
        {
            "id": 1,
            "done": false,
            "task": "finish tutorial 0",
            "due": null
        }
    ]
    ```

* Boolean search like: `curl "http://localhost:3000/todos?done=is.true" | jq`

    ```json
    [
    ]
    ```

* Select just a few fields: `curl "http://localhost:3000/todos?select=id,task" | jq` ([doc](https://postgrest.org/en/stable/api.html#vertical-filtering-columns))

    ```json
    [
        {
            "id": 1,
            "task": "finish tutorial 0",
        },
        {
            "id": 2,
            "task": "pat self on back",
        }
    ]
    ```

* Rename fields using this syntax: `NEW_NAME:field_name`: `curl "http://localhost:3000/todos?select=numéro:id,description:task" | jq` ([doc](https://postgrest.org/en/stable/api.html#renaming-columns))


    ```json
    [
        {
            "numéro": 1,
            "description": "finish tutorial 0"
        },
        {
            "numéro": 2,
            "description": "pat self on back"
        }
    ]
    ```

* Casting the datatype ([doc](https://postgrest.org/en/stable/api.html#casting-columns)):

    `curl "http://localhost:3000/todos?select=numéro:id" | jq`

    ```json
    [
        {
            "numéro": 1
        },
        {
            "numéro": 2
        }
        ]
    ```

    And now, the same but with the id as text:

    `curl "http://localhost:3000/todos?select=numéro:id::text,description:task" | jq`

    ```json
    [
        {
            "numéro": "1"
        },
        {
            "numéro": "2"
        }
        ]
    ```

* Using sorting: `curl "http://localhost:3000/todos?select=id&order=id.desc" | jq` ([doc](https://postgrest.org/en/stable/api.html#ordering))

    ```json
    [
        {
            "id": "2"
        },
        {
            "id": "1"
        }
        ]
    ```

* Limits and pagination: get the first five records (from `1` to `5`), `curl "http://localhost:3000/todos?limit=5&offset=0" | jq` then the next five (from `6` to `10`) ([doc](https://postgrest.org/en/stable/api.html#limits-and-pagination))

* Using a `OR` with two conditions: less than 18 or bigger than 21: `curl "http://localhost:3000/people?or=(age.lt.18,age.gt.21)"`

##### Call a stored procedure

Stored procedure can be accessed by using the `rpc` prefix like:

```bash
curl "http://localhost:3000/rpc/function_name"
```

More info in the [doc](https://postgrest.org/en/stable/api.html#stored-procedures).

Passing parameters:

```sql
CREATE FUNCTION api.add_them(a integer, b integer)
RETURNS integer AS $$
  SELECT a + b;
$$ LANGUAGE SQL IMMUTABLE;
```

The call below will return `3`:

```bash
curl "http://localhost:3000/rpc/add_them" \
    -X POST -H "Content-Type: application/json" \
    -d '{ "a": 1, "b": 2 }'
```

Another syntax is possible but only when the function has been created with the `IMMUTABLE` flag: `curl "http://localhost:3000/rpc/add_them?a=1&b=2"`

##### Define the output format

This can be done using the `Accept` HTTP header. 

The next command will return a csv output:

```bash
curl "http://localhost:3000/todos" \
    -H "Accept: text/csv"
```

And here just get a plain text (retrieve the title of the second task):

```bash
curl "http://localhost:3000/todos?select=task&id\=eq.2" \
    -H "Accept: text/plain"
```

##### Don't return an array

By default, an array is always returned even if there is only one result. For instance: `curl "http://localhost:3000/todos?id\=eq.2" | jq`

```json
[
  {
    "id": 2,
    "done": false,
    "task": "pat self on back",
    "due": null
  }
]
```

Since, when making the call we known we always get only one record, the array is here not needed.

Based on the [doc](https://postgrest.org/en/stable/api.html#singular-or-plural), we just need to pass `Accept: application/vnd.pgrst.object+json` header value.

```bash
curl "http://localhost:3000/todos?id\=eq.2" \
    -H "Accept: application/vnd.pgrst.object+json"| jq
```

Now, the array has disappeared. This way is certainly better for the frontend developer.

```json
{
  "id": 2,
  "done": false,
  "task": "pat self on back",
  "due": null
}
```

##### Joins

PostgREST support natively joins like f.i. 

```bash
curl "http://127.0.0.1:3000/workers?select=id,email,citizens(first_name,last_name,language_id),languages(code)&id=eq.73"
```

PostgreSQL will make the join automatically (based on the known schema) between the `workers`, `citizens` and `languages` tables.

```json
    [
        {
            "id": 73,
            "email": "john.doe@gmail.com",
            "citizens": {
                "first_name": "John",
                "last_name": "Doe",
                "language_id": 1
            },
            "languages": {
                "code": "en"
            }
        }
    ]
```

### Interact with the database with patch, post, put or delete

If we try:

```bash
curl http://localhost:3000/todos -X POST \
    -H "Content-Type: application/json" \
    -d '{"task": "do bad thing"}' | jq
```

It won't work since our anonymous user can't change things.

#### Add a trusted user

```sql
create role todo_user nologin;
grant todo_user to authenticator;

grant usage on schema api to todo_user;
grant all on api.todos to todo_user;
grant usage, select on sequence api.todos_id_seq to todo_user;
```

#### Make a secret

```bash
### Allow "tr" to process non-utf8 byte sequences
export LC_CTYPE=C

### read random bytes and keep only alphanumerics
< /dev/urandom tr -dc A-Za-z0-9 | head -c32
```

The obtained password will have a `%` at the end (on the console); **don't keep it!**.  Your password will be something like `UWY5fv357ajXUbL2gA6OuxKLGu362Q7v`.

Open `tutorial.conf` and add the key:

```text
### PASSWORD MUST BE AT LEAST 32 CHARS LONG
### add this line to tutorial.conf:

jwt-secret = "<the password you made>"
```

Make sure to shutdown the PostgREST server and rerun `./postgrest tutorial.conf`

Note: we can use plain-text password by setting the `jwt-secret-is-base64` to false:

```text
jwt-secret = "mysecretpassword"
jwt-secret-is-base64 = False
```

All configuration items can be displayed on the console with `./postgrest --example`.

The existing, used, configuration can be dumped with `./postgrest --dump-config`.

#### Sign a token

> [https://postgrest.org/en/stable/tutorials/tut1.html#step-3-sign-a-token](https://postgrest.org/en/stable/tutorials/tut1.html#step-3-sign-a-token)

Go to [https://jwt.io/#debugger-io](https://jwt.io/#debugger-io) and reuse exactly the same password (f.i. `UWY5fv357ajXUbL2gA6OuxKLGu362Q7v`).

The JSON payload data should be:

```json
{
  "name": "todo_user"
}
```

#### Make a request

Export the obtain token as below and run a POST action:

```bash
export TOKEN="<paste token here>"

curl http://localhost:3000/todos -X POST \
    -H "Authorization: Bearer $TOKEN"   \
    -H "Content-Type: application/json" \
    -d '{"task": "learn how to auth"}' | jq
```

## Laravel

> [https://g-ernaelsten.developpez.com/tutoriels/PostgREST/#LIV](https://g-ernaelsten.developpez.com/tutoriels/PostgREST/#LIV)

By adding the `symfony/http-client` reference, we can easily access to the APIs.

```bash
composer require symfony/http-client
```

```php
// j'instancie le client
$client = HttpClient::create();
// je récupère le token
$token = $client->request('POST', 'http://localhost:3000/rpc/login', [
            'body' => ["pseudo" => "jhon", "pass" => "bonjour"]
        ]);

// je crée un ensemble d'options, avec le token
$options = [
           'headers' => [
                'Prefer' => 'count=exact',
                'Authorization' => 'Bearer ' . $token->toArray()[0]['token']]
        ];
// j'appelle mon API, avec le token en options
$response = $client->request('GET', 'http://localhost:3000/film', $options);
// j'affiche le résultat sous forme de tableau
dd($response->toArray());
```

## Tips

### URL rewriting

> [https://g-ernaelsten.developpez.com/tutoriels/PostgREST/#LI-H](https://g-ernaelsten.developpez.com/tutoriels/PostgREST/#LI-H)

Without URL rewriting, we need to write things like `http://localhost:3000/task?id=eq.1001` to get that task.

To be able to use `http://localhost:3000/task/1001`, we need to use some URLs rewriting.

For nginx:

```htaccess
### support /endpoint/:id url style
location ~ ^/([a-z_]+)/([0-9]+) {

  ### make the response singular (see the `Don't return an array` chapter above)
  proxy_set_header Accept 'application/vnd.pgrst.object+json';

  proxy_pass http://localhost:3000/task?id=eq.$1;
}
```

## Hardening PostgREST

> [https://postgrest.org/en/stable/admin.html](https://postgrest.org/en/stable/admin.html)

Using a proxy, it's possible to add somes rules like DDos protection.  See [https://dev.to/apisix/a-poor-mans-api-533m#ddos-protection](https://dev.to/apisix/a-poor-mans-api-533m#ddos-protection).

We can also protect routes: see [https://dev.to/apisix/a-poor-mans-api-533m#perroute-authorization](https://dev.to/apisix/a-poor-mans-api-533m#perroute-authorization).

## Monitoring

Using [Prometheus](https://prometheus.io/) for instance: see [https://dev.to/apisix/a-poor-mans-api-533m#monitoring](https://dev.to/apisix/a-poor-mans-api-533m#monitoring).

## Implement PostgREST in an existing Docker project

Quite easy in fact. Just update your `docker-compose.yml` file and create the user in PostgreSQL.

### Add services in docker-compose.yml

Edit your `docker-compose.yml` and add something like:

```yaml
  #############
  ## postgrest #
  #############
  your_postgrest:
    container_name: your_postgrest
    image: postgrest/postgrest:latest
    ports:
      - "3000:3000"
    ## Available environment variables documented here:
    ## https://postgrest.org/en/latest/configuration.html#environment-variables
    environment:
      ## The standard connection URI format, documented at
      ## https://www.postgresql.org/docs/current/static/libpq-connect.html#LIBPQ-CONNSTRING
      - PGRST_DB_URI=postgres://${DB_USERNAME}:${DB_PASSWORD}@your_postgres:${DB_PORT}/${DB_DATABASE}
      ## The name of which database schema to expose to REST clients
      - PGRST_DB_SCHEMA=${DB_SCHEMA:-public}
      ## The database role to use when no client authentication is provided
      - PGRST_DB_ANON_ROLE=${DB_ANON_ROLE:-anon}
      ## Overrides the base URL used within the OpenAPI self-documentation hosted at the API root path
      - PGRST_OPENAPI_SERVER_PROXY_URI=http://127.0.0.1:3000
    restart: always

  ##############
  ## swagger-ui #
  ##############
  your_swagger-ui:
    container_name: your_swagger-ui
    image: swaggerapi/swagger-ui:latest
    ports:
      - "8080:8080"
    environment:
      - API_URL=http://localhost:3000/
    restart: always
```

### Create user and give permissions in your DB_DATABASE

Open a SQL console (running something like `docker exec -it YOUR_SERVICE psql -U YOUR_USER`) in your Postgres container and run:

```sql
CREATE USER anon;
GRANT USAGE ON SCHEMA public TO anon;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO anon;
GRANT SELECT ON ALL SEQUENCES IN SCHEMA public TO anon;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO anon;
```

This will create the `anon` user as specified in `docker-compose.yml` and grant him permissions to the `public` schema. If any of these two informations should be updated; also reflect the changes to `docker-compose.yml`.

## Links

* [Official documentation](https://postgrest.org/en/stable/)
* [Official Docker Hub image](https://hub.docker.com/r/postgrest/postgrest/)
* [The list of supported operators](https://postgrest.org/en/stable/api.html#operators)
* [A poor man's API](https://dev.to/apisix/a-poor-mans-api-533m)

### French tutorials

* [https://g-ernaelsten.developpez.com/tutoriels/PostgREST](https://g-ernaelsten.developpez.com/tutoriels/PostgREST)
