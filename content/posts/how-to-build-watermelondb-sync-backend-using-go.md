---
title: "How to Build Watermelondb Sync Backend Using Go"
date: 2020-03-27T19:32:09+07:00
categories: ["Go"]
tags: ["WatermelonDB", "Sync Backend"]
draft: true
---


# Init Project

```shell
$ mkdir sync-backend
$ cd sync-backend
$ go mod init sync-backend
```

<!--more-->

# Setup Database

```shell
$ docker run --name sync-backend-db -e POSTGRES_PASSWORD=dbpass123 -e POSTGRES_DB=sync_backend_db -d -p 5432:5432 postgres:12.2
```

# Setup Migration

```shell
GO111MODULE=off go get -tags 'postgres' -u -v github.com/golang-migrate/migrate/cmd/migrate
```

{{< hint warning >}}
`GO111MODULE=off` prevents `migrate` library included in `go.mod`
{{< /hint >}}

{{< hint warning >}}
If you use [`asdf`](https://github.com/asdf-vm/asdf), don't forget to run `asdf reshim golang` to locate `migrate` binary
{{< /hint >}}


# Create Tables

## `post`

```shell
$ migrate create -ext sql -dir db/migrations -seq create_post_table
```

```sql
-- db/migrations/000001_create_post_table.up.sql
CREATE TABLE post (
    id bigserial PRIMARY KEY,
    gid UUID NOT NULL,
    title TEXT NOT NULL,
    content TEXT NOT NULL,
    likes INTEGER NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

## `data_changes`

```shell
$ migrate create -ext sql -dir db/migrations -seq create_data_changes_table
```

```sql
-- db/migrations/000002_create_data_changes_table.up.sql
CREATE TABLE data_changes (
    id bigserial PRIMARY KEY,
    gid UUID NOT NULL,
    title TEXT NOT NULL,
    content TEXT NOT NULL,
    likes INTEGER NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

## Run Migrations

```shell
$ migrate -source "file://./db/migrations" -database "postgres://postgres:dbpass123@localhost:5432/sync_backend_db?sslmode=disable" up
```
