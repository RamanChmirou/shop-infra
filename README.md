# shop-infra

Infrastructure of the online shop: one `docker compose up` starts the whole system.
This repository also holds the Bruno collections that show the sales flow from the first page view to a paid order.

## Contents

- [Architecture](#architecture)
- [Repositories](#repositories)
- [Quick start](#quick-start)
- [Addresses](#addresses)
- [Users and roles](#users-and-roles)
- [Sales flow in Bruno](#sales-flow-in-bruno)
- [Logs: ELK](#logs-elk)
- [Kafka](#kafka)
- [Useful commands](#useful-commands)
- [Troubleshooting](#troubleshooting)

## Architecture

```
                  Bruno / browser
                        |
                        |  HTTPS-like REST + JWT (Keycloak token)
                        v
          +-----------------------------+        +----------+
          |  user-interaction-service   |<-----> | Keycloak |   issues and signs the tokens
          |  (the only public service)  |        +----------+
          +-----------------------------+
             |           |            |
             v           v            v
        product-     cart-         order-
        service      service       service ----- Kafka: order.placed ----> cart-service
             |           |            |
             +-----------+------------+
                         v
                    PostgreSQL (one database per service)

     every service --JSON logs--> Logstash --> Elasticsearch --> Kibana
```

- **user-interaction-service** is the only service with a published port. It checks the JWT, takes the user id
  from the token (`sub`), keeps the checkout draft and calls the internal services. A client never sends a user id.
- **product-service** is the catalog and the price calculator for configurable products.
- **cart-service** keeps the cart of a user, reprices it through product-service and closes it when the order is
  placed. Inactive carts are removed by a scheduled job.
- **order-service** turns the cart into an order, takes the payment through a (simulated) payment gateway port,
  issues the invoice (stub) and publishes `order.placed` through the transactional outbox.
- Each service has its own database. Schemas are created by Liquibase.

## Repositories

The compose file builds the services from sibling directories, so clone everything into one folder:

```
mkdir shop && cd shop
git clone https://github.com/RamanChmirou/shop-infra.git
git clone https://github.com/RamanChmirou/shop-product-service.git
git clone https://github.com/RamanChmirou/shop-cart-service.git
git clone https://github.com/RamanChmirou/shop-order-service.git
git clone https://github.com/RamanChmirou/shop-user-interaction-service.git
```

During development the work lives in the branch `all-work` of every repository (check it out in each one);
`master` contains only the initial commit until the final pull requests are merged.

## Quick start

Requirements: Docker with at least 6 GB of memory (Elasticsearch, Logstash and Kibana are the hungry ones).

```
cd shop-infra
docker compose up --build
```

The first start downloads the images and builds five services, it takes several minutes.
The system is ready when all containers are `healthy`:

```
docker compose ps
```

Start from scratch (removes all data):

```
docker compose down -v
docker compose up --build
```

## Addresses

| What | Address | Notes |
|------|---------|-------|
| Shop API (the only service entry) | http://localhost:8080 | Swagger UI: `/swagger-ui.html` |
| Keycloak | http://localhost:8180 | admin console: `admin` / `admin` |
| Kibana (logs) | http://localhost:5601 | |
| AKHQ (Kafka UI) | http://localhost:8090 | |
| PostgreSQL | localhost:5432 | `shop` / `shop` |

product, cart and order services have **no** published ports: they can be reached only through the proxy.

## Users and roles

Created by the realm import `keycloak/shop-realm.json` (realm `shop`, client `shop-web`).

| User | Password | Roles | Can |
|------|----------|-------|-----|
| `jan` | `jan123` | `USER` | browse the catalog, use the cart, check out, pay, cancel, see own orders |
| `admin` | `admin123` | `USER`, `ADMIN` | everything above and manage products, ship orders |

The catalog (`GET /api/products/**`) is public. Everything else needs a token: no token gives `401`, a token
without the needed role gives `403`. Tokens live one hour.

A token is received with the password grant (the Bruno request `00-login`):

```
POST http://localhost:8180/realms/shop/protocol/openid-connect/token
grant_type=password, client_id=shop-web, username=jan, password=jan123
```

## Sales flow in Bruno

Open the collection `bruno/user-interaction-service` in Bruno, choose the environment `local` and run the
requests in numeric order. The login request stores the token in the environment and every following request
sends it as `Authorization: Bearer ...`.

The flow: log in, browse the catalog, look at a configurable product, put it in the cart, fill in the checkout
draft (customer, delivery, payment), place the order, pay it and read the invoice. Error cases (no token,
wrong role, changed price, declined payment) follow after the happy path.

The older collections (`bruno/product-service`, `bruno/cart-service`, `bruno/order-service`) call the
internal services directly. They work only when the service ports are published, so they are kept for development.

## Logs: ELK

All services write JSON logs to Logstash when the Spring profile `docker` is active (compose sets it).

1. Open http://localhost:5601 and go to *Stack Management -> Data Views -> Create data view*.
2. Name and index pattern `shop-logs-*`, timestamp field `@timestamp`, save.
3. *Discover*: filter by service, for example `service : "order-service"`, or follow one request across
   services with the user id or the order number.

## Kafka

The topic `order.placed` is written by order-service (transactional outbox) and read by cart-service, which
closes the cart of the user. The messages can be browsed in AKHQ: http://localhost:8090.

## Useful commands

```
docker compose ps                         # state and health of every container
docker compose logs -f order-service      # logs of one service
docker compose up --build -d order-service   # rebuild and restart one service
docker compose down -v                    # stop and remove all data
```

## Troubleshooting

| Problem | Reason and fix |
|---------|----------------|
| A container is `unhealthy` right after the start | The services need up to a minute; wait and check `docker compose ps` again. |
| `401` with a valid token | The token expired (one hour) or was issued for another host. Log in again with the request `00-login` and use the address `localhost:8180`. |
| `database "..." does not exist` | `postgres/init.sql` runs only when the volume is created. Run `docker compose down -v` and start again. |
| Docker is slow or containers are killed | Give Docker at least 6 GB of memory (Settings -> Resources). |