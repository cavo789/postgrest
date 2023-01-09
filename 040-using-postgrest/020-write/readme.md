# Interact with the database with patch, post, put or delete

If we try:

```bash
curl http://localhost:3000/todos -X POST \
    -H "Content-Type: application/json" \
    -d '{"task": "do bad thing"}' | jq
```

It won't work since our anonymous user can't change things.

## Add a trusted user

```sql
create role todo_user nologin;
grant todo_user to authenticator;

grant usage on schema api to todo_user;
grant all on api.todos to todo_user;
grant usage, select on sequence api.todos_id_seq to todo_user;
```

## Make a secret

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

## Sign a token

> [https://postgrest.org/en/stable/tutorials/tut1.html#step-3-sign-a-token](https://postgrest.org/en/stable/tutorials/tut1.html#step-3-sign-a-token)

Go to [https://jwt.io/#debugger-io](https://jwt.io/#debugger-io) and reuse exactly the same password (f.i. `UWY5fv357ajXUbL2gA6OuxKLGu362Q7v`).

The JSON payload data should be:

```json
{
  "name": "todo_user"
}
```

## Make a request

Export the obtain token as below and run a POST action:

```bash
export TOKEN="<paste token here>"

curl http://localhost:3000/todos -X POST \
    -H "Authorization: Bearer $TOKEN"   \
    -H "Content-Type: application/json" \
    -d '{"task": "learn how to auth"}' | jq
```
