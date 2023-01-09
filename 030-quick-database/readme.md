# Set-up a sample database for this tutorial

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

## Create the tutorial.conf file

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
