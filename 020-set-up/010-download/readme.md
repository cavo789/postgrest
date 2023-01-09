# Set-up

## Postgres

Get and run a postgres database server: 

```bash
docker run --name tutorial -p 5433:5432 \
    -e POSTGRES_PASSWORD=mysecretpassword \
    -d postgres
```

## PostgREST

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
