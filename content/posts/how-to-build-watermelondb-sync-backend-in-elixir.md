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

<!--more-->

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
$ mix phx.gen.schema Blog.Post posts title:string content:string likes:integer deleted_at:utc_datetime_usec version:integer version_created:integer --binary-id
```

Edit `xxx_create_posts.exs`

```diff
# priv/repo/migrations/xxx_create_posts.exs
defmodule BlogApp.Repo.Migrations.CreatePosts do
  use Ecto.Migration

  def change do
    create table(:posts, primary_key: false) do
      add :id, :binary_id, primary_key: true
      add :title, :string
      add :content, :string
      add :likes, :integer
      add :deleted_at, :utc_datetime_usec
-     add :version, :integer
+     add :version, :bigint, default: fragment("nextval('version_seq')")
-     add :version_created, :integer
+     add :version_created, :bigint, default: fragment("nextval('version_seq')")
-     timestamps()
+     timestamps([type: :utc_datetime_usec])
    end

  end
end
```

Edit `post.ex`.
Cast: not `:version`
Validate: not `:likes, :deleted_at, :version`

```diff
# lib/blog_app/blog/post.ex
defmodule BlogApp.Blog.Post do
  use Ecto.Schema
  import Ecto.Changeset

  @primary_key {:id, :binary_id, autogenerate: true}
  @foreign_key_type :binary_id
+  @derive {Jason.Encoder, only: [:id, :title, :content, :likes]}
  schema "posts" do
    field :content, :string
    field :deleted_at, :utc_datetime_usec
    field :likes, :integer
    field :title, :string
    field :version, :integer
    field :version_created, :integer

-   timestamps()
+   timestamps([type: :utc_datetime_usec])
  end

  @doc false
  def changeset(post, attrs) do
    post
-   |> cast(attrs, [:title, :content, :likes, :deleted_at, :version, :version_created])
+   |> cast(attrs, [:title, :content, :likes, :deleted_at])
-   |> validate_required([:title, :content, :likes, :deleted_at, :version, :version_created])
+   |> validate_required([:title, :content])
  end
end
```

```shell
$ mix ecto.migrate
```

## Sync Endpoint

Sync endpoint will be handled by:

- push: `POST /api/sync?lastPulledVersion=<lastPulledVersion>`
- pull: `GET /api/sync?lastPulledVersion=<lastPulledVersion>`

Edit `lib/blog_app_web/router.ex`

```elixir
# lib/blog_app_web/router.ex
defmodule BlogAppWeb.Router do
    # ...
    scope "/api", BlogAppWeb do
        pipe_through :api
        post "/sync", SyncController, :push
        get "/sync", SyncController, :pull
    end
    # ...
end
```

Create `lib/blog_app_web/controllers/sync_controller.ex`

```elixir
# lib/blog_app_web/controllers/sync_controller.ex

```

## Push

Create context `BlogApp.Sync`

```elixir
# lib/blog_app/sync.ex
defmodule BlogApp.Sync do
  alias BlogApp.{Repo, Blog}

  def push(changes, last_pulled_version) do
    Ecto.Multi.new()
    |> Blog.record_posts(changes["posts"], last_pulled_version)
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

  def record_posts(%Multi{} = multi, post_changes, last_pulled_version) do
    case check_conflict_version_posts(post_changes, last_pulled_version) do
      :no_conflict ->
        multi
        |> record_created_posts(post_changes["created"])
        |> record_updated_posts(post_changes["updated"])
        |> record_deleted_posts(post_changes["deleted"])

      _ -> {:error, "conflict version"}
    end
  end

  # ...
end
```

### Check Conflict

`check_conflict_version_posts/2` implementation.

```elixir
# lib/blog_app/blog.ex
defmodule BlogApp.Blog do
  # ...
  def check_conflict_version_posts(post_changes, last_pulled_version) do
    ids =
      Enum.concat(post_changes["created"], post_changes["updated"])
      |> Enum.map(fn post -> post.id end)
      |> Enum.concat(post_changes["deleted"])

    count =
      Post
      |> select([p], count(p.version))
      |> where([p], p.id in ^ids)
      |> where([p], p.version > ^last_pulled_version or p.version_created > ^last_pulled_version)
      |> Repo.one()

    case count do
      0 -> :no_conflict
      _ -> :conflict
    end
  end
  # ...
end
```


### Storing Record Changes 

`record_created_posts/2` & `record_updated_posts/2` implementation.
`upsert_posts/3` handle both create & update case.

```elixir
# lib/blog_app/blog.ex
defmodule BlogApp.Blog do
  # ...
  def record_created_posts(%Multi{} = multi, created_changes),
    do: upsert_posts(multi, :create_posts, created_changes)

  def record_updated_posts(%Multi{} = multi, updated_changes),
    do: upsert_posts(multi, :update_posts, updated_changes)

  def upsert_posts(%Multi{} = multi, _name, attrs) when is_nil(attrs),
    do: multi

  def upsert_posts(%Multi{} = multi, name, attrs) do
    now = DateTime.utc_now()

    data =
      attrs
      |> Enum.map(fn row ->
        row
        |> Map.put(:inserted_at, now)
        |> Map.put(:updated_at, now)
      end)

    Multi.insert_all(multi, name, Post, data,
      conflict_target: :id,
      on_conflict: {:replace_all_except, [:id, :version_created, :inserted_at, :deleted_at]},
      returning: true
    )
  end
  # ...
end
```

`record_deleted_posts/2` implementation

```elixir
# lib/blog_app/blog.ex
defmodule BlogApp.Blog do
  # ...
  def record_deleted_posts(%Multi{} = multi, deleted_ids) when is_nil(deleted_ids),
    do: multi

  def record_deleted_posts(%Multi{} = multi, deleted_ids) do
    now = DateTime.utc_now()

    query = Post |> where([p], p.id in ^deleted_ids)

    Ecto.Multi.update_all(multi, :delete_posts, query, set: [deleted_at: now])
  end
end
```

## Pull

```elixir
# lib/blog_app/sync.ex
defmodule BlogApp.Sync do
  # ...

  def pull(last_pulled_version) do
    %{latest_version: latest_version_posts, changes: posts_changes} =
      Blog.list_posts_changes(last_pulled_version)

    latest_version =
      [latest_version_posts]
      |> Enum.max()

    %{
      "latestVersion" => latest_version,
      "changes" => %{
        "posts" => posts_changes
      }
    }
  end
end
```

Let's implement `Blog.list_posts_changes/1`

```elixir
# lib/blog_app/blog.ex
defmodule BlogApp.Blog do
  # ...
  def list_posts_changes(last_pulled_version) do
    posts_latest =
      Post
      |> where([p], p.version_created > ^last_pulled_version or p.version > ^last_pulled_version)
      |> Repo.all()

    posts_changes =
      posts_latest
      |> Enum.group_by(fn post ->
        cond do
          post.version_created > last_pulled_version and is_nil(post.deleted_at) -> :created
          post.inserted_at != post.updated_at and is_nil(post.deleted_at) -> :updated
          not is_nil(post.deleted_at) -> :deleted
        end
      end)
      |> Map.update(:deleted, [], fn posts -> posts |> Enum.map(fn post -> post.id end) end)

    latest_version =
      posts_latest
      |> Enum.flat_map(fn post -> [post.version, post.version_created] end)
      |> Enum.max(fn -> 0 end)

    %{latest_version: latest_version, changes: posts_changes}
  end
end
```