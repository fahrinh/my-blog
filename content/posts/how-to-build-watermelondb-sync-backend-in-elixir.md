---
title: "How to Build WatermelonDB Sync Backend in Elixir"
date: 2020-04-24T21:27:12+07:00
categories: ["Elixir"]
tags: ["Phoenix", "WatermelonDB", "Sync Backend"]
draft: true
summary: WatermelonDB is a reactive database for React frontend application that supports data synchronization. This tutorial explains the concept and shows step by step how to build sync backend using Phoenix (Elixir)
---

TODO:
- Table explaining columns on client & db. LocalDB: column, ServerDB: column, description, on create, on update, on delete.
- Seq Diagram: 1. Existing WatermelonDB flow 2. Proposed solution. 'People' = Client



# WatermelonDB

[WatermelonDB](https://github.com/Nozbe/WatermelonDB) is a reactive database for React frontend application that supports data synchronization.

![WatermelonDB](http://www.plantuml.com/plantuml/proxy?src=https://raw.githubusercontent.com/fahrinh/my-blog/master/diagram/watermelondb.plantuml)

What I like about this database is you can bring your own sync backend (HTTP-based) as long as it complies with this spec:

| Operation |       Request Params / Body        |          Response Body          |
|-----------|:----------------------------------:|:-------------------------------:|
| Pull      |          - `lastPulledAt`: _integer, Unix time in milliseconds (ms)_         | - `changes`: _(JSON)_ <br/> - `timestamp`: _integer, Unix time in milliseconds (ms)_ |
| Push      | - `changes`: _(JSON)_ <br/> - `lastPulledAt`: _integer, Unix time in milliseconds (ms)_ |               _X_               |

The following is a brief how pull and push operation works. Please refer to [Sync documentation](https://nozbe.github.io/WatermelonDB/Advanced/Sync.html) for the details

## Pull Operation

Request:

- `lastPulledAt` is a timestamp retrieved in the last/previous pull operation

Response:

- `changes` is a JSON containing changes of data (created, updated, deleted) since `lastPulledAt` at server
- `timestamp` is a timestamp that will replace `lastPulledAt` for the next pull operation.

## Push Operation

Request:

- `changes` is a JSON containing data changes on the client (local) that will be applied by server on server DB.
- `lastPulledAt` is a timestamp retrieved in the last/previous pull operation. This is for conflict detection. Server compare modification time of each row of `changes` on server DB with `lastPulledAt`. If it is greater, there is a conflict.

Response:

- No specified response

## `changes` Example

```json
{
    "posts": {
        "created": [
            {
                "id": "d1633195-156f-4f9d-9ccf-7740203b080e",
                "_status": "created",
                "_changed": "",
                "title": "Phoenix",
                "content": "Phoenix is a web framework for Elixir",
                "likes": 200,
                "created_at": 1588400731806,
                "updated_at": 1588400731806
            }
        ],
        "updated": [
            {
                "id": "2d7c6a82-eb04-47b1-be52-6f8f6cf806ff",
                "_status": "updated",
                "_changed": "updated_at,title,content,likes",
                "title": "Elixir",
                "content": "Elixir is amazing",
                "likes": 100,
                "created_at": 1588389279195,
                "updated_at": 1588400691047
            }
        ],
        "deleted": [
            "2b130e52-079d-4b31-9f42-ce257cf546f0"
        ]
    },
    "comments": {
        "created": [],
        "updated": [
            {
                "id": "1e945c88-baf2-4db7-aa39-286b6865b3fb",
                "comment": "That's good!"
            }
        ],
        "deleted": []
    }
}
```

- `created` and `updated` is an array of object containing created / updated records
- `deleted` is an array of string of deleted IDs

## Sync Flow

Based on the documentation and [sync code example (`synchronize()`) on client side]((https://nozbe.github.io/WatermelonDB/Advanced/Sync.html#using-synchronize)), this is what will be expected from sync backend:

![WatermelonDB Sync Flow](http://www.plantuml.com/plantuml/proxy?src=https://raw.githubusercontent.com/fahrinh/my-blog/master/diagram/watermelondb-sync-flow.plantuml)

# Proposed Alternative Sync Approach

While in iterations of prototyping sync backend, I took another approach for tracking changes and made a workaround for an issue in regard to WatermelonDB sync behaviour on client side.

## Using Auto-incrementing Counter (Version) for Tracking Changes

WatermelonDB sync documentation is good enough to gives a tips for implementing sync backend by using timestamp. However, it also states:

  > For extra safety, we recommend adding a MySQL/PostgreSQL procedure that will ensure last_modified uniqueness and monotonicity (will increment it by one if a record with this last_modified or greater already exists in the database).

  > This protects against **weird edge cases related to server clock time changes (NTP time sync, leap seconds, etc.)** (**Alternatively, instead of using timestamps, you could use auto-incrementing couters, but you'd have to ensure they are consistent across the whole database, not just one table**)

I completely agree with this.
A dependency on server clock time reliability for data changes is a problem.
The changes of data have to be recorded in *correct order*.
Out-of-order records will be a major headache if that bug happened in production.

I followed its suggestion to use auto-incrementing counter. In my approach, for tracking changes, it needs these server DB setup: a global sequence (`version_seq`) & each table have columns: `version` (int), `version_created` (int), and `deleted_at` (timestamp).  A new data will have values of `version` and `version_created` generated by `version_seq`. If that data is updated, value of `version` is also updated with value generated by `version_seq` (`version_created` value not changed). If the data is deleted, it simply set current time as a value for `deleted_at` (new or updated data has null value for `deleted_at`).


## Workaround for Sync on Client Side

There is a problem if we use [sync code example (`synchronize()`) for client side on the documentation](https://nozbe.github.io/WatermelonDB/Advanced/Sync.html#using-synchronize).

Please look again at diagram <a href="#sync-flow">Sync Flow</a> above.

In **5**, after WatermelonDB receives `changes` & `timestamp`, internally, `timestamp` value is set as `lastPulledAt` for the next pull operation. That is not problem.

At first, I assume it will have same mechanism for push operation.
We got new `timestamp` that will be become the next `lastPulledAt`.
But I am wrong.
Look at **11**.
No response at all for push operation AND no way to explicitly set `lastPulledAt` on the push operation.
It means for the next pull operation, we will get changes that we've just pushed on previous push operation. Well, it actually mentioned [in the documentation](https://nozbe.github.io/WatermelonDB/Advanced/Sync.html#current-limitations):

 > Current limitations
 
 > 2. During next sync pull, changes we've just pushed will be pulled again, which is unnecessary. It would be better if server, during push, also pulled local changes since `lastPulledAt` and responded with NEW timestamp to be treated as `lastPulledAt`.


I don't know why this library designed to behave like this. It raised [an issue complaining/questioning about that](https://github.com/Nozbe/WatermelonDB/issues/649).
So this is what I did for a temporary solution/workaround:



# Application Example: Blog App



# Sync Backend Implementation
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

  # ...
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

      post "/sync/push", SyncController, :push
      get "/sync/pull", SyncController, :pull
    end
    # ...
end
```

### Controller

Create `lib/blog_app_web/controllers/sync_controller.ex`

```elixir
# lib/blog_app_web/controllers/sync_controller.ex
defmodule BlogAppWeb.SyncController do
  use BlogAppWeb, :controller
  alias BlogApp.Sync

  def push(
        %Plug.Conn{
          body_params: req_body,
          query_params: %{"lastPulledVersion" => last_pulled_version}
        } = conn,
        _params
      ) do

    resp =  Sync.push(req_body, String.to_integer(last_pulled_version))
    json(conn, resp)
  end

  def pull(
        %Plug.Conn{
          query_params: %{"lastPulledVersion" => last_pulled_version}
        } = conn,
        _params
      ) do

    resp = Sync.pull(String.to_integer(last_pulled_version))
    json(conn, resp)
  end
end
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
    |> case do
      {:ok, %{get_latest_version_posts: latest_version_posts}} ->
        latest_version =
          [last_pulled_version, latest_version_posts]
          |> Enum.max()

        %{ "latestVersion" => latest_version }
    end
  end

  # ...
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
    multi
    |> Multi.run(:check_conflict_posts, fn _, _changes ->
      case check_conflict_version_posts(post_changes, last_pulled_version) do
        :no_conflict -> {:ok, :no_conflict}
        :conflict -> {:error, :conflict}
      end
    end)
    |> record_created_posts(post_changes["created"])
    |> record_updated_posts(post_changes["updated"])
    |> record_deleted_posts(post_changes["deleted"])
    |> Multi.run(:get_latest_version_posts, fn _, prev_results ->
      %{
        create_posts: {_, created_posts},
        delete_posts: {_, deleted_posts},
        update_posts: {_, updated_posts}
      } = prev_results

      latest_version = find_latest_version(created_posts ++ updated_posts ++ deleted_posts)

      {:ok, latest_version}
    end)
  end

  # ...

  defp find_latest_version(posts) do
    posts
    |> Enum.flat_map(fn post -> [post.version, post.version_created] end)
    |> Enum.max(fn -> 0 end)
  end
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
      |> Enum.map(fn post -> post["id"] end)
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
        |> Map.put("inserted_at", now)
        |> Map.put("updated_at", now)
        |> Map.take(["id", "title", "content", "likes", "inserted_at", "updated_at"])
        |> key_to_atom()
      end)

    Multi.insert_all(multi, name, Post, data,
      conflict_target: :id,
      on_conflict: {:replace_all_except, [:id, :version_created, :inserted_at, :deleted_at]},
      returning: true
    )
  end

  def key_to_atom(map) do
    Enum.reduce(map, %{}, fn
      {key, value}, acc when is_atom(key) -> Map.put(acc, key, value)
      {key, value}, acc when is_binary(key) -> Map.put(acc, String.to_existing_atom(key), value)
    end)
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
  
    data =
      deleted_ids
      |> Enum.map(fn id ->
        %{id: id, deleted_at: now}
      end)

    Multi.insert_all(multi, :delete_posts, Post, data,
      conflict_target: :id,
      on_conflict: {:replace, [:deleted_at, :version]},
      returning: true
    )
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
      [last_pulled_version, latest_version_posts]
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
      |> Map.update(:created, [], fn posts -> posts end)
      |> Map.update(:updated, [], fn posts -> posts end)
      |> Map.update(:deleted, [], fn posts -> posts |> Enum.map(fn post -> post.id end) end)

    latest_version = find_latest_version(posts_latest)

    %{latest_version: latest_version, changes: posts_changes}
  end
end
```