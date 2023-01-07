# PostgREST tutorial

![Banner](./banner.svg)

**Personnal learning** How to use [https://postgrest.org/en/stable/tutorials/tut0.html](https://postgrest.org/en/stable/tutorials/tut0.html)


## Downloads

### Postgres

Get and run a postgres server: 

```bash
docker run --name tutorial -p 5433:5432 \
    -e POSTGRES_PASSWORD=mysecretpassword \
    -d postgres
```

### PostgREST

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

### Making somes calls

* Get the entire table: `curl http://localhost:3000/todos | jq`
* Only a given record (`id=2`): `curl http://localhost:3000/todos\?id\=eq.2 | jq`

### Debugging

* Get the list of all tables for the `api` schema: `SELECT * FROM information_schema.tables WHERE table_schema='api';`
