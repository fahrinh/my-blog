---
title: "How to Build WatermelonDB Sync Backend in Elixir"
date: 2020-04-24T21:27:12+07:00
categories: ["Elixir", "Phoenix"]
tags: ["WatermelonDB", "Sync Backend"]
draft: false
images : ["images/watermelondb.png"]
summary: WatermelonDB is a reactive database for React frontend application that supports data synchronization. This tutorial explains the concept and shows step by step how to build sync backend using Phoenix (Elixir)
---

# WatermelonDB

[WatermelonDB](https://github.com/Nozbe/WatermelonDB) is a reactive database for React frontend application that supports data synchronization.

![WatermelonDB](https://www.plantuml.com/plantuml/proxy?src=https://raw.githubusercontent.com/fahrinh/my-blog/master/diagram/watermelondb.plantuml)

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

Based on the documentation and [sync code example (`synchronize()`) on client side](https://nozbe.github.io/WatermelonDB/Advanced/Sync.html#using-synchronize), this is what will be expected from sync backend:

![WatermelonDB Sync Flow](https://www.plantuml.com/plantuml/proxy?src=https://raw.githubusercontent.com/fahrinh/my-blog/master/diagram/watermelondb-sync-flow.plantuml)

# Proposed Alternative Sync Approach

While in iterations of prototyping sync backend, I took another approach for tracking changes and made a workaround for an issue in regard to WatermelonDB sync behaviour on client side.

## Using Auto-incrementing Counter (Version) + Timestamp for Tracking Changes

WatermelonDB sync documentation is good enough to gives a tips for implementing sync backend by using timestamp. It also states:

  > This protects against weird edge cases related to server clock time changes (NTP time sync, leap seconds, etc.) (**Alternatively, instead of using timestamps, you could use auto-incrementing couters, but you'd have to ensure they are consistent across the whole database, not just one table**)

I followed its suggestion to use auto-incrementing counter. In my approach, for tracking changes, it needs these server DB setup: a global sequence (`version_seq`) & each table have columns: `version` (int), `version_created` (int), `created_at_server` (timestamp), `updated_at_server` (timestamp), and `deleted_at_server` (timestamp).

Please see <a href="#database-design">Database Design</a> to know how this is implemented in practice.

## Workaround for Sync on Client Side

There is a problem if we use [sync code example (`synchronize()`) for client side on the documentation](https://nozbe.github.io/WatermelonDB/Advanced/Sync.html#using-synchronize).

Please look again at <a href="#sync-flow">Sync Flow diagram</a> above.

In **8**, after WatermelonDB receives `changes` & `timestamp`, internally, `timestamp` value is set as `lastPulledAt` for the next pull operation. That is not problem.

At first, I assume it will have same mechanism for push operation.
We got new `timestamp` that will be become the next `lastPulledAt`.
But I am wrong.
Look at **15**.
No response at all for push operation AND no way to explicitly set `lastPulledAt` on the push operation.
It means for the next pull operation, we will get changes that we've just pushed on previous push operation. Well, it actually mentioned [in the documentation](https://nozbe.github.io/WatermelonDB/Advanced/Sync.html#current-limitations):

 > Current limitations
 
 > 2. During next sync pull, changes we've just pushed will be pulled again, which is unnecessary. It would be better if server, during push, also pulled local changes since `lastPulledAt` and responded with NEW timestamp to be treated as `lastPulledAt`.


I don't know why this library designed to behave like this. It raised [an issue complaining/questioning about that](https://github.com/Nozbe/WatermelonDB/issues/649).

### Sync Flow Workaround

So this is what I did for a temporary solution/workaround:

![WatermelonDB Sync Flow Workaround](https://www.plantuml.com/plantuml/proxy?src=https://raw.githubusercontent.com/fahrinh/my-blog/master/diagram/watermelondb-sync-flow-workaround.plantuml)

- introduce variable `latestVersionOfSession` & `changesOfSession` (**1**) 
- call `synchronize()` twice (**2** & **20**)
- on first `synchronize()`, pull & push operation retrieve `latestVersion` & `changes` (**8** & **19**) then set it as `latestVersionOfSession` & `changesOfSession` value
- on second `synchronize()`, pull operation only set `lastPulledAt = latestVersionOfSession` & `changes = changesOfSession` to be applied on LocalDB (**22**). Push operation does nothing.

This is workaround for the client side. The code is available on [the next post]({{< ref "building-an-offline-first-react-web-app-using-watermelondb-in-phoenix-elixir.md" >}}).

# Application Example: BlogApp

Let's say we want to build a blog app (web based) that supports data synchronization.
User can submit, edit, and delete a post content. If user click Sync button, data located on current browser will be synced to server. So if user open another browser (another client device), data will be automically synced and available on that browser.

This tutorial only covers how to build sync backend implementation. Frontend (ReactJS) implementation is available on the next post: [Building an Offline First React Web App Using WatermelonDB in Phoenix (Elixir)]({{< ref "building-an-offline-first-react-web-app-using-watermelondb-in-phoenix-elixir.md" >}}).

## Database Design

<table>
<thead>
  <tr>
    <th colspan="2">LocalDB</th>
    <th colspan="3">ServerDB</th>
  </tr>
  <tr>
    <th colspan="2">WatermelonDB</br><code>posts</code> table</th>
    <th colspan="3">PostgreSQL</br><code>posts</code> table</th>
  </tr>
  <tr>
    <th>Column</th>
    <th>Type</th>
    <th>Column</th>
    <th>Type</th>
    <th>Default</th>
  </tr>
</thead>
<tbody>
  <tr align="center">
    <td><code>id</code></td>
    <td>string (UUID format)</td>
    <td><code>id</code></td>
    <td>uuid (binary)</td>
    <td>-</td>
  </tr>
  <tr align="center">
    <td><code>title</code></td>
    <td>string</td>
    <td><code>title</code></td>
    <td>varchar</td>
    <td>-</td>
  </tr>
  <tr align="center">
    <td><code>content</code></td>
    <td>string</td>
    <td><code>content</code></td>
    <td>varchar</td>
    <td>-</td>
  </tr>
  <tr align="center">
    <td><code>likes</code></td>
    <td>number</td>
    <td><code>likes</code></td>
    <td>integer</td>
    <td>-</td>
  </tr>
  <tr align="center">
    <td><code>created_at</code></td>
    <td>number</br>(UNIX timestamp in ms)</td>
    <td><code>created_at</code></td>
    <td>timestamp (in μs)</td>
    <td>-</td>
  </tr>
  <tr align="center">
    <td><code>updated_at</code></td>
    <td>number</br>(UNIX timestamp in ms)</td>
    <td><code>updated_at</code></td>
    <td>timestamp (in μs)</td>
    <td>-</td>
  </tr>
  <tr align="center">
    <td>-</td>
    <td>-</td>
    <td><code>created_at_server</code></td>
    <td>timestamp (in μs)</td>
    <td>-</td>
  </tr>
  <tr align="center">
    <td>-</td>
    <td>-</td>
    <td><code>updated_at_server</code></td>
    <td>timestamp (in μs)</td>
    <td>-</td>
  </tr>
  <tr align="center">
    <td>-</td>
    <td>-</td>
    <td><code>deleted_at_server</code></td>
    <td>timestamp (in μs)</td>
    <td>-</td>
  </tr>
  <tr align="center">
    <td>-</td>
    <td>-</td>
    <td><code>version</code></td>
    <td>bigint</td>
    <td><code>nextval('version_seq')</code></td>
  </tr>
  <tr align="center">
    <td>-</td>
    <td>-</td>
    <td><code>version_created</code></td>
    <td>bigint</td>
    <td><code>nextval('version_seq')</code></td>
  </tr>
</tbody>
</table>

## Determining Data Changes

For push operation:

- when a new data is created:
  - `version = nextval('version_seq')`
  - `version_created = nextval('version_seq')`
  - `created_at_server = current time`
  - `updated_at_server = current time`
- when a data is updated:
  - `version = nextval('version_seq')`
  - `updated_at_server = current time`
- when a data is deleted:
  - `version = nextval('version_seq')`
  - `deleted_at_server = current time`

For pull operation:

- retrieve all data that were changed since `lastPulledVersion`:

  `SELECT * FROM posts WHERE version_created > <lastPulledVersion> OR version > <lastPulledVersion>`
- Then categorize which records were created, updated, or deleted :
  - created

    `version_created > <lastPulledVersion> AND deleted_at_server IS NULL`

  - updated

    `created_at_server != updated_at_server AND deleted_at_server IS NULL`

  - deleted

    `deleted_at_server IS NOT NULL`

# Sync Backend Implementation

Sync Backend consists of four main components: `SyncController`, `Sync` context, `Blog` context, and `Repo`. If you come from another framework, _context_ is kind of like _service_.

![BlogApp Architecture](https://www.plantuml.com/plantuml/proxy?src=https://raw.githubusercontent.com/fahrinh/my-blog/master/diagram/blog-app-architecture.plantuml)

We will build sync backend using Elixir 1.10 and Phoenix 1.5.1

Install Phoenix 1.5.1:

```shell
$ mix archive.uninstall phx_new
$ mix archive.install hex phx_new 1.5.1
```

Generate a new Phoenix web app:

```shell
$ mix phx.new blog_app
```

We will use PostgreSQL on Docker with password and database name specified on `config/dev.exs`:

```shell
$ docker run --name blog-db -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=blog_app_dev -d -p 5432:5432 postgres:12.2
```

Create the database:

```shell
$ cd blog_app
$ mix ecto.create
```

Create `version_seq` sequence that will generate version for each data changes :

```shell
$ mix ecto.gen.migration create_version_seq
```

```elixir
# priv/repo/migrations/xxx_create_version_seq.exs
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

Generate `Post` schema:

```shell
$ mix phx.gen.schema Blog.Post posts title:string content:string likes:integer push_id:integer created_at:utc_datetime_usec updated_at:utc_datetime_usec created_at_server:utc_datetime_usec updated_at_server:utc_datetime_usec deleted_at_server:utc_datetime_usec version:integer version_created:integer --binary-id
```

Set default value of `version*` columns with a incremental number generated by `version_seq`.
As it may increase overtime, change `version*` columns type to `bigint` to support bigger value.
We don't need `inserted_at` and `updated_at` columns generated by Phoenix so we omit `timestamps()`.
Edit `xxx_create_posts.exs`:

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
      add :created_at, :utc_datetime_usec
      add :updated_at, :utc_datetime_usec
      add :created_at_server, :utc_datetime_usec
      add :updated_at_server, :utc_datetime_usec
      add :deleted_at_server, :utc_datetime_usec
      add :push_id, :integer
-     add :version, :integer
+     add :version, :bigint, default: fragment("nextval('version_seq')")
-     add :version_created, :integer
+     add :version_created, :bigint, default: fragment("nextval('version_seq')")

-     timestamps()
    end
  end
end
```

To enable JSON encoding for `Post` schema, annotate it with `@derive` `Jason.Encoder` for certain columns only:

```diff
# lib/blog_app/blog/post.ex
defmodule BlogApp.Blog.Post do
  use Ecto.Schema
  import Ecto.Changeset

  @primary_key {:id, :binary_id, autogenerate: true}
  @foreign_key_type :binary_id
+ @derive {Jason.Encoder, only: [:id, :title, :content, :likes]}
  schema "posts" do
    field :title, :string
    field :content, :string
    field :likes, :integer
    field :created_at, :utc_datetime_usec
    field :updated_at, :utc_datetime_usec
    field :created_at_server, :utc_datetime_usec
    field :updated_at_server, :utc_datetime_usec
    field :deleted_at_server, :utc_datetime_usec
    field :push_id, :integer
    field :version, :integer
    field :version_created, :integer

-   timestamps()
  end
  # ...
end
```

```shell
$ mix ecto.migrate
```

## Sync Endpoint

Sync endpoint will be handled by:

- push: `POST /api/sync/push?lastPulledVersion=<lastPulledVersion>`
- pull: `GET /api/sync/pull?lastPulledVersion=<lastPulledVersion>`

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

Changes have to be recorded in a DB transaction.
If there is a failed data operation, every operation must be rolled back.

In a push operation, data that will be recorded are also annotated with a `push_id`.
Later on, after push operation has been successfully applied, all changes since `last_pulled_version` are retrieved to become push response except those have already been applied (filtered with `push_id`).

`BlogApp.Sync.push/2`:

```elixir
# lib/blog_app/sync.ex
defmodule BlogApp.Sync do
  alias BlogApp.{Repo, Blog}

  def push(changes, last_pulled_version) do
    push_id = Enum.random(1..1_000_000_000)

    Ecto.Multi.new()
    |> Blog.record_posts(changes["posts"], last_pulled_version, push_id)
    |> Repo.transaction()

    pull(last_pulled_version, push_id)
  end
  # ...
end
```

`BlogApp.Blog.record_posts/4`:

```elixir
# lib/blog_app/blog.ex
defmodule BlogApp.Blog do
  import Ecto.Query
  alias Ecto.Multi
  alias BlogApp.Repo
  alias BlogApp.Blog.Post

  def record_posts(%Multi{} = multi, post_changes, last_pulled_version, push_id) do
    multi
    |> Multi.run(:check_conflict_posts, fn _, _changes ->
      case check_conflict_version_posts(post_changes, last_pulled_version) do
        :no_conflict -> {:ok, :no_conflict}
        :conflict -> {:error, :conflict}
      end
    end)
    |> record_created_posts(post_changes["created"] |> set_push_id(push_id))
    |> record_updated_posts(post_changes["updated"] |> set_push_id(push_id))
    |> record_deleted_posts(post_changes["deleted"], push_id)
  end
  # ...
  defp set_push_id(posts, push_id) do
    posts
    |> Enum.map(fn post -> post |> Map.put("push_id", push_id) end)
  end
  # ...
end
```

### Conflict Detection

Conflict happens when other users/clients have modified data that we're pushing.

`Blog.check_conflict_version_posts/2` :

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

Data changes are saved on database in bulk using `INSERT INTO CONFLICT` on PostgreSQL.
This is also known as UPSERT (update or insert).

`Blog.upsert_posts/3` handle both create & update case.
`Blog.record_created_posts/2` & `Blog.record_updated_posts/2` :

```elixir
# lib/blog_app/blog.ex
defmodule BlogApp.Blog do
  # ...
  def record_created_posts(%Multi{} = multi, created_changes),
    do: upsert_posts(multi, :create_posts, created_changes)

  def record_updated_posts(%Multi{} = multi, updated_changes),
    do: upsert_posts(multi, :update_posts, updated_changes)

  def upsert_posts(%Multi{} = multi, _name, changes) when is_nil(changes),
    do: multi

  def upsert_posts(%Multi{} = multi, name, changes) do
    now = DateTime.utc_now()

    posts =
      changes
      |> Enum.map(fn row ->
        row
        |> Map.put("created_at", row["created_at"] * 1000 |> DateTime.from_unix!(:microsecond))
        |> Map.put("updated_at", row["updated_at"] * 1000 |> DateTime.from_unix!(:microsecond))
        |> Map.put("created_at_server", now)
        |> Map.put("updated_at_server", now)
        |> Map.take(["id", "title", "content", "likes", "created_at", "updated_at", "created_at_server", "updated_at_server", "push_id"])
        |> key_to_atom()
      end)

    Multi.insert_all(multi, name, Post, posts,
      conflict_target: :id,
      on_conflict: {:replace_all_except, [:id, :version_created, :created_at_server, :deleted_at_server]},
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

`Blog.record_deleted_posts/3` :

```elixir
# lib/blog_app/blog.ex
defmodule BlogApp.Blog do
  # ...
  def record_deleted_posts(%Multi{} = multi, deleted_ids, _push_id) when is_nil(deleted_ids),
    do: multi

  def record_deleted_posts(%Multi{} = multi, deleted_ids, push_id) do
    now = DateTime.utc_now()

    posts =
      deleted_ids
      |> Enum.map(fn id ->
        %{id: id, deleted_at_server: now, push_id: push_id}
      end)

    Multi.insert_all(multi, :delete_posts, Post, posts,
      conflict_target: :id,
      on_conflict: {:replace, [:deleted_at_server, :version, :push_id]},
      returning: true
    )
  end
  # ...
end
```

## Pull

Pull endpoint calls `Sync.pull` without `push_id` specified.
It means all data changes since `last_pulled_version` become the response of pull operation.

```elixir
# lib/blog_app/sync.ex
defmodule BlogApp.Sync do
  # ...
  def pull(last_pulled_version, push_id \\ nil) do
    %{latest_version: latest_version_posts, changes: posts_changes} =
      Blog.list_posts_changes(last_pulled_version, push_id)

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

`Blog.list_posts_changes/2`:

```elixir
# lib/blog_app/blog.ex
defmodule BlogApp.Blog do
  # ...
  def list_posts_changes(last_pulled_version, push_id) do
    posts_latest =
      Post
      |> where([p], p.version_created > ^last_pulled_version or p.version > ^last_pulled_version)
      |> Repo.all()

    posts_changes =
      posts_latest
      |> Enum.reject(fn post -> is_just_pushed(post, push_id) end)
      |> Enum.group_by(fn post ->
        cond do
          post.version_created > last_pulled_version and is_nil(post.deleted_at_server) -> :created
          post.created_at_server != post.updated_at_server and is_nil(post.deleted_at_server) -> :updated
          not is_nil(post.deleted_at_server) -> :deleted
        end
      end)
      |> Map.update(:created, [], fn posts -> posts end)
      |> Map.update(:updated, [], fn posts -> posts end)
      |> Map.update(:deleted, [], fn posts -> posts |> Enum.map(fn post -> post.id end) end)

    latest_version = find_latest_version(posts_latest)

    %{latest_version: latest_version, changes: posts_changes}
  end

  defp find_latest_version(posts) do
    posts
    |> Enum.flat_map(fn post -> [post.version, post.version_created] end)
    |> Enum.max(fn -> 0 end)
  end
  # ...
  defp is_just_pushed(_post, push_id) when is_nil(push_id), do: false
  defp is_just_pushed(post, push_id), do: post.push_id == push_id
end
```

## Run the Backend

```shell
$ mix phx.server
```

Sync endpoint will be available on 
- push: `POST http://localhost:4000/api/sync/push?lastPulledVersion=<lastPulledVersion>`
- pull: `GET http://localhost:4000/api/sync/pull?lastPulledVersion=<lastPulledVersion>`



# What's Next ?

We have build a sync backend for WatermelonDB frontend application.
On the next post, we will continue to code the frontend application with sync capability: [Building an Offline First React Web App Using WatermelonDB in Phoenix (Elixir)]({{< ref "building-an-offline-first-react-web-app-using-watermelondb-in-phoenix-elixir.md" >}})