# quacklake

A Rust project for interacting with DuckLake.

## Project Plan

For more details on the project plan, see the [project plan](docs/plan.md).

## Development

### Commit Messages

This project follows the semantic commit message format. For more details, see the [commit message guidelines](docs/dev/commit.md).

## Development Setup

This project includes a Docker Compose setup for local development. It starts a LocalStack container with S3 enabled and a PostgreSQL container for metadata storage.

To start the development environment, run:

```bash
docker-compose up
```

This will make the following services available:

-   **LocalStack (S3)**: `http://localhost:4566`
-   **PostgreSQL**: `localhost:5432` (user: `quacklake`, password: `quacklake`, db: `ducklake_metadata`)

## License

This project is licensed under the MIT License.
