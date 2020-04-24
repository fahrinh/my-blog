---
title: "How to Build WatermelonDB Sync Backend in Elixir"
date: 2020-04-24T21:27:12+07:00
categories: ["Elixir"]
tags: ["WatermelonDB", "Sync Backend"]
draft: true
---

First and foremost, this tutorial will use Phoenix 1.5.1

```shell
$ mix archive.uninstall phx_new
$ mix archive.install hex phx_new 1.5.1
```

```shell
$ mix phx.new blog_app
```

```shell
$ docker run --name blog-db -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=blog_app_dev -d -p 5432:5432 postgres:12.2
```

```shell
$ cd blog_app
$ mix ecto.create
```

```shell
$ mix ecto.gen.migration create_version_seq
```

```elixir
defmodule BlogApp.Repo.Migrations.CreateVersionSeq do
  use Ecto.Migration

  def change do
    execute "CREATE SEQUENCE version_seq"
  end
end
```

```shell
$ mix ecto.migrate
```

```shell
$ mix phx.gen.schema Blog.Post posts title:string content:string likes:integer deleted_at:datetime version:integer --binary-id
```

Edit `xxx_create_posts.exs`

```elixir
add :version, :integer, default: fragment("nextval('version_seq')")
```

Edit `post.ex`.
Cast: not `:version`
Validate: not `:likes, :deleted_at, :version`

```elixir
def changeset(post, attrs) do
    post
    |> cast(attrs, [:title, :content, :likes, :deleted_at])
    |> validate_required([:title, :content])
end
```

```shell
$ mix ecto.migrate
```

Create context `BlogApp.Sync`

```elixir
# lib/blog_app/sync.ex
defmodule BlogApp.Sync do
  alias BlogApp.{Repo, Blog}

  def push(changes) do
    Ecto.Multi.new()
    |> Blog.record_post(changes["post"])
    |> Repo.transaction()
  end
end
```

Let's implement `record_posts/2` on `BlogApp.Blog` context

```elixir
# lib/blog_app/blog.ex
defmodule BlogApp.Blog do
  import Ecto.Query
  alias Ecto.Multi
  alias BlogApp.Repo
  alias BlogApp.Blog.Post

  def record_posts(%Multi{} = multi, post_changes) do
    multi
    |> record_created_posts(post_changes["created"])
    |> record_updated_posts(post_changes["updated"])
    |> record_deleted_posts(post_changes["deleted"])
  end

  # ...
end
```
