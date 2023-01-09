# Read data from the database

From now, we can try it:

```bash
curl http://localhost:3000/todos | jq
```

(if `jq` isn't installed yet, please run `sudo apt install jq`)

That call will return the list of all todos defined in the `todos` table as a JSON string.

## Making some calls

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

### Call a stored procedure

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

### Define the output format

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

### Don't return an array

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

### Joins

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
