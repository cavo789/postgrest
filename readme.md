# PostgREST tutorial

**Personnal learning** How to use [https://postgrest.org/en/stable/tutorials/tut0.html](https://postgrest.org/en/stable/tutorials/tut0.html)

![Banner](./banner.svg)

> A docker-compose example with Postgres, PostREST and Swagger UI: [https://github.com/mattddowney/compose-postgrest](https://github.com/mattddowney/compose-postgrest)

## Downloads

### Postgres

Get and run a postgres server: 

```bash
docker run --name tutorial -p 5433:5432 \
    -e POSTGRES_PASSWORD=mysecretpassword \
    -d postgres
```

### PostgREST

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

#### PostgREST - The manual way

If you need it manually, here is how to proceed:

* Download the binary from https://github.com/PostgREST/postgrest/releases/download/v10.1.1/postgrest-v10.1.1-linux-static-x64.tar.xz
* Run `tar xJf postgrest-v10.1.1-linux-static-x64.tar.xz`

## Create the database

### Set-up DB

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

### Ready for querying with anonymous user, read-only

From now, we can try it:

```bash
curl http://localhost:3000/todos | jq
```

(if `jq` isn't installed yet, please run `sudo apt install jq`)

### Create a privileged user with write permissions

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
# Allow "tr" to process non-utf8 byte sequences
export LC_CTYPE=C

# read random bytes and keep only alphanumerics
< /dev/urandom tr -dc A-Za-z0-9 | head -c32
```

The obtained password will have a `%` at the end (on the console); **don't keep it!**.  Your password will be something like `UWY5fv357ajXUbL2gA6OuxKLGu362Q7v`.

Open `tutorial.conf` and add the key:

```text
# PASSWORD MUST BE AT LEAST 32 CHARS LONG
# add this line to tutorial.conf:

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

### Somes usefull links

* The list of operators we can use like equal, less than, is true (for boolean fields), ... [https://postgrest.org/en/stable/api.html#operators](https://postgrest.org/en/stable/api.html#operators). Note: it is probably better to use a view for complicated filtering options.

    ```sql
    CREATE VIEW fresh_stories AS
    SELECT *
    FROM stories
    WHERE pinned = true
        OR published > now() - interval '1 day'
    ORDER BY pinned DESC, published DESC;
    ```

    ```bash
    curl "http://localhost:3000/fresh_stories"
    ````

### Making somes calls

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

#### Call a stored procedure

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

#### Define the output format

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

#### Don't return an array

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

Now, the array has dissappeared. This way is certainly better for the frontend developer.

```json
{
  "id": 2,
  "done": false,
  "task": "pat self on back",
  "due": null
}
```

#### More samples

* Using a `OR` with two conditions: less than 18 or bigger than 21: `curl "http://localhost:3000/people?or=(age.lt.18,age.gt.21)"`

### Debugging

* Get the list of all tables for the `api` schema: `SELECT * FROM information_schema.tables WHERE table_schema='api';`
