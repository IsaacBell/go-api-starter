# Go API Starter

A production-oriented starter for Go REST APIs. Opinionated project layout, batteries included: Echo for HTTP routing, PostgreSQL for persistence, RabbitMQ for messaging, InfluxDB for time-series business events, Prometheus for metrics, and JWT/OAuth2 for auth.

Clone it, rename the module, and start adding domains.

## Stack

| Concern | Choice |
| --- | --- |
| HTTP | [Echo](https://echo.labstack.com/) v4 |
| Database | PostgreSQL 15 via [go-pg](https://github.com/go-pg/pg) v9 |
| Messaging | RabbitMQ (streadway/amqp) |
| Time-series | InfluxDB 2.x |
| Metrics | Prometheus (+ echo-prometheus) |
| Logging | zerolog (+ lecho) |
| Auth | JWT, OAuth2, zxcvbn password strength |
| API docs | Swagger (swaggo/echo-swagger) |
| RPC/schema | Protocol Buffers |
| Validation | go-playground/validator |

## Requirements

- Go 1.20+
- Docker and Docker Compose

The stack runs PostgreSQL, InfluxDB, and RabbitMQ alongside the app, so it has high RAM requirements. You will want a strong machine or VM.

`start-server.sh` installs `protobuf-compiler` via `apt`, so it assumes Debian/Ubuntu (or WSL). On macOS, install protobuf with `brew install protobuf` and run the Docker steps by hand.

## Setup

Configure your environment before running the app. To start from the default config:

```bash
cp .env.example .env
```

## Running the app

```bash
bash start-server.sh
```

This builds the `soapstone-image` container, brings the Compose stack up, installs the protobuf toolchain, and builds the Go binary. Kill it with Ctrl+C as normal.

## Accessing tools

Connect to PostgreSQL:

```bash
docker exec -it -u postgres postgresDB psql
```

Connect to InfluxDB: browse to `http://localhost:8086` (credentials come from the `INFLUXDB_*` variables in your `.env`).

## Project structure

1. The root directory contains things not related to code directly — Docker Compose, CI/CD, readme, bash scripts.

2. The `cmd` package contains code for starting applications (main packages). The directory name for each application should match the name of the executable you want. An application may produce multiple binaries, so we follow the Go convention of placing the main package as a subdirectory of `cmd`. In a scheduler application, for example, the binary would live under `cmd/cron`. It also loads configuration and passes it to the service initializers.

3. The rest of the code lives under `/pkg`, which contains `utl` and microservice directories.

4. Microservice directories such as `api` (naming corresponds to `cmd/` naming) contain a folder per domain it interacts with: `user`, `car`, `appointment`, and so on.

5. Domain directories such as `user` contain all application/business logic plus two additional directories, `platform` and `transport`.

6. `platform` contains packages providing support for databases, authentication, and marshaling. Most packages under `platform` are decoupled behind interfaces. Every platform gets its own package: `postgres`, `elastic`, `redis`, `memcache`.

7. `transport` contains HTTP handlers. The package receives requests, marshals and validates them, then passes them to the corresponding service.

8. `utl` contains helper packages and models — mock, middleware, configuration, server.

## Adding CRUD for a new table

Say you have a `cars` table for employees' cars. To implement CRUD on it:

1. Inside `pkg/utl/model`, create `car.go`. Put your entity (struct) there, along with any methods on it.

2. Create a `car` folder in the (micro)service where your service will live, most likely under `api`. Inside, create the service and its test (`car/car.go` and `car/car_test.go`). You can test without writing a single query by mocking the database logic in `/mock/mockdb`. If you have complex queries touching other entities, add files like `car_users.go` or `car_templates.go` here.

3. Inside the `car` folder, create `platform`, `transport`, and `logging`.

4. Code interacting with a platform such as PostgreSQL goes under `car/platform/pgsql` (`pkg/api/car/platform/pgsql/car.go`).

5. In `pkg/api/car/transport`, create `http.go` — this is where your handlers live. Add `http_test.go` alongside it to test the API.

6. In `logging`, create `car.go` and copy the logic from another service. This handles request/response logging.

7. In `pkg/api/api.go`, wire everything up: instantiate the car service, then pass it to the logging and transport services.

## Implementing other platforms

Similar to implementing APIs that rely only on a database:

1. In the service package, in `car.go`, add an interface corresponding to the platform — `Indexer`, `Reporter`, etc.

2. The rest of the procedure is the same, except that under `/platform` you create a new folder for your platform, for example `elastic`.

3. Once the platform logic is implemented, create an instance of it in `main.go` (for example `elastic.Client`) and pass it as an argument to the car service (`pkg/api/car/car.go`).

## Running database queries in a transaction

Before interacting with the database, create a new transaction:

```go
err := s.db.RunInTransaction(func(tx *pg.Tx) error {
    // Application service here
})
```

Instead of passing the database client as `s.db`, pass `tx` inside this function. Handle the error accordingly.

## Contributing

We loosely follow a trunk-based protocol.

## License

See [LICENSE](LICENSE).