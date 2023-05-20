## Image

```
version: "3.7"
volumes:
  kong_data: {}

networks:
  kong-net:
services:
  #######################################
  # Postgres: The database used by Kong
  #######################################
  kong-database:
    image: postgres:9.6
    container_name: kong-postgres
    restart: on-failure
    networks:
      - kong-net
    volumes:
      - kong_data:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: kong
      POSTGRES_PASSWORD: ${KONG_PG_PASSWORD:-kong}
      POSTGRES_DB: kong
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "kong"]
      interval: 30s
      timeout: 30s
      retries: 3
  #######################################
  # Kong database migration
  #######################################
  kong-migration:
    image: ${KONG_DOCKER_TAG:-kong:latest}
    command: kong migrations bootstrap
    networks:
      - kong-net
    restart: on-failure
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-database
      KONG_PG_DATABASE: kong
      KONG_PG_USER: kong
      KONG_PG_PASSWORD: ${KONG_PG_PASSWORD:-kong}
    depends_on:
      - kong-database
  #######################################
  # Kong: The API Gateway
  #######################################
  kong:
    image: ${KONG_DOCKER_TAG:-kong:latest}
    restart: on-failure
    networks:
      - kong-net
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-database
      KONG_PG_DATABASE: kong
      KONG_PG_USER: kong
      KONG_PG_PASSWORD: ${KONG_PG_PASSWORD:-kong}
      KONG_PROXY_LISTEN: 0.0.0.0:8000
      KONG_PROXY_LISTEN_SSL: 0.0.0.0:8443
      KONG_ADMIN_LISTEN: 0.0.0.0:8001
    depends_on:
      - kong-database
    healthcheck:
      test: ["CMD", "kong", "health"]
      interval: 10s
      timeout: 10s
      retries: 10
    ports:
      - "8000:8000"
      - "8001:8001"
      - "8443:8443"
      - "8444:8444"
  #######################################
  # Konga database prepare
  #######################################
  konga-prepare:
    image: pantsel/konga:latest
    command: "-c prepare -a postgres -u postgresql://kong:${KONG_PG_PASSWORD:-kong}@kong-database:5432/konga"
    networks:
      - kong-net
    restart: on-failure
    depends_on:
      - kong-database
  #######################################
  # Konga: Kong GUI
  #######################################
  konga:
    image: pantsel/konga:latest
    restart: always
    networks:
      - kong-net
    environment:
      DB_ADAPTER: postgres
      DB_URI: postgresql://kong:${KONG_PG_PASSWORD:-kong}@kong-database:5432/konga
      NODE_ENV: production
    depends_on:
      - kong-database
    ports:
      - "1337:1337"
```

---

## Step

```

  docker-compose up -d

```

## Usage

```

  UI:
    {{domain_name}}:1337


  kong_admin:
    {{domain_name}}:8001


  kong_client:
    {{domain_name}}:8000
```
