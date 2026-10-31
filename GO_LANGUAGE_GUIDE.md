# Go in This Project

This guide explains Go features you can see in this codebase. Each section shows a small example and translates it into plain English.

> The examples are based on the project, with some details shortened for readability.

## 1. Packages and imports

Every Go file starts by naming its package:

```go
package orders
```

A **package** groups related Go files. For example, files under `internal/orders` belong to the `orders` package.

Files import other packages when they need their code:

```go
import (
    "context"
    "fmt"

    repo "github.com/sikozonpc/ecom/internal/adapters/postgresql/sqlc"
)
```

The `repo` before the import path is an **alias**. It lets this package refer to the imported package as `repo`, for example `repo.Order`.

Go uses capitalization to control visibility:

```go
func NewService(...) {} // exported: usable from another package
func validate(...) {}  // unexported: usable only inside this package
```

The same rule applies to types, fields, and variables. `Service` is exported; `svc` is not.

## 2. Variables and `:=`

The short declaration operator `:=` creates a variable and figures out its type:

```go
conn, err := pgx.Connect(ctx, cfg.db.dsn)
```

This creates two variables: `conn` for the connection and `err` for a possible error. You can also declare a variable with an explicit type:

```go
var err error
```

Go is statically typed: once a variable has a type, it cannot be assigned a value of an unrelated type.

## 3. Structs: grouping data

A **struct** is a named group of fields:

```go
type orderItem struct {
    ProductID int64 `json:"productId"`
    Quantity  int32 `json:"quantity"`
}
```

This says an order item contains a product ID and a quantity. The text in backticks is a **struct tag**. Here it tells Go's JSON package to use the JSON property names `productId` and `quantity`.

Struct values can be created by naming their fields:

```go
item := orderItem{
    ProductID: 42,
    Quantity:  2,
}
```

Naming fields makes it easier to read and safer if the struct's field order changes.

## 4. Pointers: referring to a value

A pointer holds the address of a value. In Go, `*T` means “pointer to a `T`,” and `&value` means “address of this value.”

```go
var count int
address := &count // & obtains the address of count
*address = 3       // * follows the address and updates count
```

The project uses pointers for long-lived objects such as the database connection and service implementations. It also passes a pointer to a struct when the JSON decoder needs to fill that struct:

```go
var request createOrderParams
err := json.Read(r, &request)
```

Without `&request`, the decoder would receive a copy of the value rather than the address where it should put the decoded fields.

### Pointer receivers

The `*` before a receiver type means the method operates on a pointer to the service:

```go
func (s *svc) ListProducts(ctx context.Context) ([]repo.Product, error) {
    return s.repo.ListProducts(ctx)
}
```

Read this as: “`ListProducts` is a method on a pointer to `svc`.” This is common for services and lets methods use the same service instance without copying it.

Not every `*` is a pointer. In `60 * time.Second`, it means multiplication.

## 5. Methods and interfaces

A **method** is a function attached to a type. The receiver is the value the method belongs to:

```go
func (s *svc) ListProducts(ctx context.Context) ([]repo.Product, error) {
    return s.repo.ListProducts(ctx)
}
```

An **interface** describes behavior by listing methods:

```go
type Service interface {
    ListProducts(ctx context.Context) ([]repo.Product, error)
}
```

Go interfaces are satisfied **implicitly**. A type does not need to declare “I implement `Service`.” If it has the required method with the matching signature, it satisfies the interface.

The product service constructor returns the interface:

```go
func NewService(repo repo.Querier) Service {
    return &svc{repo: repo}
}
```

The handler can depend on `Service` rather than the concrete `svc` type. This keeps the handler less tied to one implementation and makes it easier to substitute a test implementation.

## 6. Slices and `range`

A **slice** is a flexible list of values. The `[]` in this return type means “a slice of products”:

```go
([]repo.Product, error)
```

The order request uses a slice for its items:

```go
Items []orderItem `json:"items"`
```

The service loops over the items with `range`:

```go
for _, item := range tempOrder.Items {
    // use item
}
```

`range` visits each element. The blank identifier `_` means “ignore this result”; here, the code does not need the item's position in the slice.

`len` returns the number of items:

```go
if len(tempOrder.Items) == 0 {
    return repo.Order{}, fmt.Errorf("at least one item is required")
}
```

## 7. Errors are values

Most Go functions that can fail return an `error` as one of their results:

```go
conn, err := pgx.Connect(ctx, dsn)
if err != nil {
    return err
}
```

This is the standard Go pattern: check the error, then decide what to do. It is not an exception that automatically jumps to another part of the program.

The order service also defines named errors:

```go
var ErrProductNotFound = errors.New("product not found")
```

These give the application recognizable error values for specific situations. In Go, `error` is an interface; functions such as `fmt.Errorf` create values that implement it.

## 8. `context.Context`

`context.Context` carries request-related information and cancellation signals through function calls:

```go
func (s *svc) ListProducts(ctx context.Context) ([]repo.Product, error) {
    return s.repo.ListProducts(ctx)
}
```

The HTTP handler passes `r.Context()` to the service. If the HTTP request is cancelled or times out, database operations that use that context can stop too.

## 9. `defer`: run cleanup when a function exits

`defer` schedules a function call to run when the surrounding function returns:

```go
defer conn.Close(ctx)
```

This is useful for cleanup. The connection is closed whether the function returns normally or after an error.

The order service uses the same idea for a transaction:

```go
defer tx.Rollback(ctx)
```

That schedules rollback as a safety net. If the transaction has already been committed, the later rollback does not undo that commit.

## 10. Functions as values

The health endpoint passes an anonymous function (a function without its own name) to the router:

```go
r.Get("/health", func(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("all good"))
})
```

The router keeps this function and calls it when a matching request arrives. The function can also **close over** values from the surrounding function, meaning it can use those values without listing them as parameters.

## 11. Building a value from smaller pieces

The `application` struct groups the app's configuration and database connection:

```go
type application struct {
    config config
    db     *pgx.Conn
}
```

The startup code builds it by assigning fields:

```go
api := application{
    config: cfg,
    db:     conn,
}
```

Then it passes the application to methods such as `mount` and `run`. This is a simple form of **dependency wiring**: startup creates shared things (like the database connection) and gives them to the parts that need them.

## 12. `panic` versus ordinary errors

The project checks a database connection error with `if err != nil` and calls `panic(err)` if connecting fails:

```go
conn, err := pgx.Connect(ctx, cfg.db.dsn)
if err != nil {
    panic(err)
}
```

`panic` stops normal execution and starts unwinding the current goroutine. It is different from returning an ordinary error. For application operations, Go code usually prefers returning and handling errors explicitly; startup code sometimes treats a required dependency failing as fatal.

## Quick reference

| Go syntax | Beginner translation |
|---|---|
| `type Product struct { ... }` | Define a data shape |
| `[]Product` | A list (slice) of products |
| `*pgx.Conn` | A pointer to a database connection |
| `&value` | Get the address of a value |
| `func (s *svc) Do()` | A method on a pointer to `svc` |
| `type Service interface { ... }` | Describe a set of required methods |
| `if err != nil` | Check whether something failed |
| `defer cleanup()` | Run cleanup when this function returns |
| `for _, item := range items` | Visit each item in a list |
| `ctx context.Context` | Pass cancellation/request information along |

## A good reading order

To follow a request through this project, read:

1. `cmd/api.go` — routes and application wiring
2. `internal/orders/handlers.go` — HTTP request decoding and response
3. `internal/orders/types.go` — request data and service interface
4. `internal/orders/service.go` — validation and database work
5. `internal/adapters/postgresql/sqlc/queries.sql.go` — generated SQL calls

The existing [PROJECT_EXPLAINED.md](PROJECT_EXPLAINED.md) explains the overall architecture; this guide focuses on the Go language features used to build it.
