# Implement PostgREST in an existing Docker project

Quite easy in fact. Just update your `docker-compose.yml` file and create the user in PostgreSQL.

## Add services in docker-compose.yml

Edit your `docker-compose.yml` and add something like:

```yaml
  #############
  # postgrest #
  #############
  your_postgrest:
    container_name: your_postgrest
    image: postgrest/postgrest:latest
    ports:
      - "3000:3000"
    # Available environment variables documented here:
    # https://postgrest.org/en/latest/configuration.html#environment-variables
    environment:
      # The standard connection URI format, documented at
      # https://www.postgresql.org/docs/current/static/libpq-connect.html#LIBPQ-CONNSTRING
      - PGRST_DB_URI=postgres://${DB_USERNAME}:${DB_PASSWORD}@your_postgres:${DB_PORT}/${DB_DATABASE}
      # The name of which database schema to expose to REST clients
      - PGRST_DB_SCHEMA=${DB_SCHEMA:-public}
      # The database role to use when no client authentication is provided
      - PGRST_DB_ANON_ROLE=${DB_ANON_ROLE:-anon}
      # Overrides the base URL used within the OpenAPI self-documentation hosted at the API root path
      - PGRST_OPENAPI_SERVER_PROXY_URI=http://127.0.0.1:3000
    restart: always

  ##############
  # swagger-ui #
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

## Create user and give permissions in your DB_DATABASE

Open a SQL console (running something like `docker exec -it YOUR_SERVICE psql -U YOUR_USER`) in your Postgres container and run:

```sql
CREATE USER anon;
GRANT USAGE ON SCHEMA public TO anon;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO anon;
GRANT SELECT ON ALL SEQUENCES IN SCHEMA public TO anon;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO anon;
```

This will create the `anon` user as specified in `docker-compose.yml` and grant him permissions to the `public` schema. If any of these two informations should be updated; also reflect the changes to `docker-compose.yml`.
