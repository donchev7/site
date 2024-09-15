---
date: 2023-01-14
image: 'post/2023/pg-gopher.png'
title: 'Working with PostgreSQL in Go using pgx'
slug: 'working-with-postgresql-in-go-using-pgx'
toc: true
tags:
  - Go
  - PostgreSQL
---

## Introduction

Having had the chance recently to work with SQL again after years of NoSQL (DynamoDB, MongoDB, Redis, CosmosDB), I was surprised to see how much I missed it.

The Project I am working on uses Go and even tough Go has a couple of good ORMs ([ent](https://entgo.io/)) and bad ones ([gorm](https://gorm.io/)), I decided to go with the native PostgreSQL driver [pgx](https://github.com/jackc/pgx). Why? Because I needed all the performance I could get and pgx provides some nice features like:

- [prepared statements](https://www.postgresql.org/docs/15/sql-prepare.html)
- [connection pooling](https://www.postgresql.org/docs/15/libpq-pool.html)
- binary decoding - pgx can decode query results directly into Go structs, which can be faster and more convenient than manually parsing rows.
- feature rich - pgx supports a wide range of PostgreSQL features, including notifications, large objects, and COPY.

The only downside is the somewhat lack of documentation. I had to go through GitHub issues and read the source code to understand the API. Having invested the time to learn it, I thought it would be nice to share my experience.

In this post I'll write down some of the things I learned while working with pgx. I will try to keep it as simple as possible, but I will also try to cover some of the more advanced features.

Specifically, I will cover the following topics:

- Connection pool setup
- Inserts
- Bulk inserts
- Querying and parsing results into Go structs


## Connection pool setup

Setting up a connection pool is the recommended way when we are working with DBs. Instead of creating a new connection for each query a connection pool will reuse connections. To set up a connection pool I need to call `pgxpool.New` and pass it a connection string: 

```go
package pg

import (
	"context"
	"fmt"
	"sync"

	"github.com/jackc/pgx/v5/pgxpool"
)

type postgres struct {
	db *pgxpool.Pool
}

var (
	pgInstance *postgres
	pgOnce     sync.Once
)

func NewPG(ctx context.Context, connString string) (*postgres, error) {
	pgOnce.Do(func() {
		db, err := pgxpool.New(ctx, connString)
		if err != nil {
			return fmt.Errorf("unable to create connection pool: %w", err)
		}

		pgInstance = &postgres{db}
	})

	return pgInstance, nil
}

func (pg *postgres) Ping(ctx context.Context) error {
	return pg.db.Ping(ctx)
}

func (pg *postgres) Close() {
	pg.db.Close()
}

```

Note that I am using a singleton pattern to make sure that I only have one connection pool. This is not strictly necessary, but it is a good practice to avoid creating multiple connection pools.

Also note that I like to put my database connection code in a separate package. This way I can reuse it in other projects.

## Inserting data

We will cover three ways of inserting data. Single row inserts, bulk inserts and COPY inserts.

### Single row inserts

Inserting a single row is pretty straightforward. Having initialized a connection pool, we can call `Exec` on it and pass it a SQL query and the values we want to insert:

```go

func (pg *postgres) InsertUser(ctx context.Context) error {
  query := `INSERT INTO users (name, email) VALUES (@userName, @userEmail)`
  args := pgx.NamedArgs{
    "userName": "Bobby",
    "userEmail": "bobby@donchev.is",
  }
  _, err := pg.db.Exec(ctx, query, args)
  if err != nil {
    return fmt.Errorf("unable to insert row: %w", err)
  }

  return nil
}
```

Note that I am using named arguments instead of positional arguments (e.g. `$1`, `$2`). This is a good practice because it makes the code more readable and it is also more secure. If you use positional arguments, you can easily make a mistake and pass the arguments in the wrong order. With named arguments, you can't make this mistake.

### Bulk inserts

Bulk inserts are a bit more complicated. You need to create a `pgx.Batch` and add all the queries you want to execute to it. Then you need to call `SendBatch` on the connection pool and pass it the batch. This will return a `pgx.BatchResults` object. You can then iterate over the results and check for errors:

```go
func (pg *postgres) BulkInsertUsers(ctx context.Context, users []model.User) error {
  query := `INSERT INTO users (name, email) VALUES (@userName, @userEmail)`

  batch := &pgx.Batch{}
  for _, user := range users {
    args := pgx.NamedArgs{
      "userName": user.Name,
      "userEmail": user.Email,
    }
    batch.Queue(query, args)
  }
  
  results := pg.db.SendBatch(ctx, batch)
  defer results.Close()

  for _, user := range users {
    _, err := results.Exec()
    if err != nil {
      var pgErr *pgconn.PgError
      if errors.As(err, &pgErr) && pgErr.Code == pgerrcode.UniqueViolation {
          log.Printf("user %s already exists", user.Name)
          continue
      }

      return fmt.Errorf("unable to insert row: %w", err)
    }
  }

  return results.Close()
}
```

This code is a bit more involved but I think it shows a couple nice to knows like how to check for a specific error code and how to use the pgx.Batch struct.

The `SendBatch` method returns a `pgx.BatchResults` struct. This struct has a `Exec` method that is somewhat  misleading. It doesn't actually execute the insert, it just returns the result. I can call `Exec` multiple times to get the results of all the inserts in the batch. The `Close` method on the `pgx.BatchResults` struct is what actually executes the batch. Its idempotent, so I can call it multiple times.



### COPY inserts

Bulk inserts a performant way to insert data, but they are not the fastest. If you want to insert a lot of data, you should use the COPY method.

We can do a COPY insert with the `CopyFrom` method:

```go
func (pg *postgres) CopyInsertUsers(ctx context.Context, users []model.User) error {
  entries := [][]any{}
  columns := []string{"name", "email"}
  tableName := "user"

  for _, user := range users {
    entries = append(entries, []any{user.Name, user.Email})
  }

  _, err := pg.db.CopyFrom(
    ctx,
    pgx.Identifier{tableName},
    columns,
    pgx.CopyFromRows(entries),
  )

  if err != nil {
    return fmt.Errorf("error copying into %s table: %w", tableName, err)
  }

  return nil
}
```

That isn't too bad =)

When to use COPY and when to use bulk inserts really for me boils down to wether I need to know if a particular insert failed or not. If I don't need to know, I will use COPY. If I do need to know, I will use bulk inserts.

## Querying data

pgx has a couple of methods for querying data. The most basic one is `QueryRow`. This method will return a `pgx.Row` struct. This struct has a `Scan` method that will scan the result into a struct:

```go
func (pg *postgres) GetUsers(ctx context.Context) ([]model.User, error) {
  query := `SELECT name, email FROM user LIMIT 10`
  
  rows, err := pg.db.Query(ctx, query)
  if err != nil {
    return nil, fmt.Errorf("unable to query users: %w", err)
  }
  defer rows.Close()

  users := []model.User{}
  for rows.Next() {
    user := model.User{}
    err := rows.Scan(&user.Name, &user.Email)
    if err != nil {
      return nil, fmt.Errorf("unable to scan row: %w", err)
    }
    users = append(users, user)
  }

  return users, nil
}
```

However, what I found out is that since v5 we have generics and can do this instead:

```go
func (pg *postgres) GetUser(ctx context.Context, name string) ([]model.User, error) {
  query := `SELECT name, email FROM user LIMIT 10`

  rows, err := pg.db.Query(ctx, query)
  if err != nil {
    return nil, fmt.Errorf("unable to query users: %w", err)
  }
  defer rows.Close()

  return pgx.CollectRows(rows, pgx.RowToStructByName[model.User])
}
```

This is a lot simpler and it also has the advantage of not having to know the column names in advance. It will just scan the result into the struct.

Doing SQL in Go got a lot of hate in the past because of `interface{}` and manually scanning the result into a struct. But with pgx v5, this is no longer the case. I think that libraries like [sqlx](https://github.com/jmoiron/sqlx) and [scany](https://github.com/georgysavva/scany) are great but not necessary anymore.


## Conclusion

We covered a lot of ground in this post. We looked at how to connect to Postgres using `pgx.Pool`, three different ways how to insert data, how to query and parse data. We also looked at how to use the new pgx v5 features like named arguments and the `pgx.CollectRows` method.

I hope you enjoyed this post and that you learned something new. If you have any questions or remarks, feel free to ping me on twitter [@bobby_donchev](https://twitter.com/bobby_donchev).

Thanks.