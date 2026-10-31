# Ecommerce API

This project is a small Go-based e-commerce backend API. It exposes product listing and order-creation endpoints backed by PostgreSQL.

For a beginner-friendly walkthrough with diagrams and architecture notes, see [PROJECT_EXPLAINED.md](PROJECT_EXPLAINED.md).

## Running the API

```bash
docker compose up -d
```

## Prerequisites
- Docker
- [Goose](https://github.com/pressly/goose)
- [SQLC](https://docs.sqlc.dev/en/latest/)

## Running the API

```bash
go run cmd/*.go
```

## Adding a new table
1. Create a new migration file under `internal/adapters/postgres/migrations`
```bash
goose -s create create_products sql
```

2. Run the migrations
```bash
goose up
```

3. Generate the SQLC code
```bash
sqlc generate
```

4. Consume it from your services 🚀