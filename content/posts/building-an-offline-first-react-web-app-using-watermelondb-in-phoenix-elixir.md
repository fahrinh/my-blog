---
title: "Building an Offline First React Web App Using WatermelonDB in Phoenix (Elixir)"
date: 2020-04-29T10:58:39+07:00
categories: ["React", "Phoenix"]
tags: ["WatermelonDB", "Offline First App"]
draft: true
---

We will build a web application that supports local db synced with remote db. This post is a continuation from the previous post: [How to Build WatermelonDB Sync Backend in Elixir]({{< ref "how-to-build-watermelondb-sync-backend-in-elixir.md" >}}).

<!--more-->

## Installing Dependencies

```shell
$ cd assets
$ npm i react react-dom uuid @nozbe/watermelondb @nozbe/with-observables
$ npm i -D @babel/preset-react @babel/plugin-proposal-decorators @babel/plugin-proposal-class-properties @babel/plugin-transform-runtime
```

Configure `assets/.babelrc`:

```js
{
    "presets": [
        "@babel/preset-env", "@babel/preset-react"
    ],
    "plugins": [
        ["@babel/plugin-proposal-decorators", { "legacy": true }],
        ["@babel/plugin-proposal-class-properties", { "loose": true }],
        [
          "@babel/plugin-transform-runtime",
           {
             "helpers": true,
             "regenerator": true
           }
        ]
      ]
}
```


## Let's Code

Remove entirely `lib/blog_app_web/templates/page/index.html.eex` and replace it with this code:

```html
<div id="blog-app" />
```

All frontend code related to blog app will be located in `assets/js/blog`.
So, we will import entry point of blog app in `assets/js/app.js`:

```js
// assets/js/app.js
import "../css/app.scss"
import "phoenix_html"
import "./blog"
```

## Setup `Database`

```react
// assets/js/blog/index.js
import React from 'react';
import { render } from 'react-dom';
import { Database } from '@nozbe/watermelondb'
import LokiJSAdapter from '@nozbe/watermelondb/adapters/lokijs'
import DatabaseProvider from '@nozbe/watermelondb/DatabaseProvider'

import schema from './model/schema'
import Post from './model/Post'

import App from './App'

const adapter = new LokiJSAdapter({
  schema,
  useWebWorker: false,
  useIncrementalIndexedDB: true,
  onIndexedDBVersionChange: () => {
    if (checkIfUserIsLoggedIn()) {
      window.location.reload()
    }
  },
})

const database = new Database({
  adapter,
  modelClasses: [
    Post,
  ],
  actionsEnabled: true,
})

const rootElement = document.getElementById('blog-app');

render(
  <DatabaseProvider database={database}>
    <App />
  </DatabaseProvider>,
  rootElement);
```

### Schema

```react
// assets/js/blog/model/schema.js
import { appSchema, tableSchema } from '@nozbe/watermelondb'

const mySchema = appSchema({
    version: 1,
    tables: [
        tableSchema({
            name: 'posts',
            columns: [
                { name: 'title', type: 'string' },
                { name: 'content', type: 'string' },
                { name: 'likes', type: 'number' },
                { name: 'created_at', type: 'number' },
                { name: 'updated_at', type: 'number' },
            ]
        })
    ]
})

export default mySchema
```

### Model

```react
// assets/js/blog/model/Post.js
import { Model } from '@nozbe/watermelondb'
import { field, date, readonly } from '@nozbe/watermelondb/decorators'

export default class Post extends Model {
    static table = 'posts'

    @field('title') title
    @field('content') content
    @field('likes') likes
    @readonly @date('created_at') createdAt
    @readonly @date('updated_at') updatedAt
}
```

## `App.js`

![BlogApp Mockup](http://www.plantuml.com/plantuml/proxy?src=https://raw.githubusercontent.com/fahrinh/my-blog/master/diagram/blog-app-mockup.plantuml)

`<App/>` consists of:

- `<PostForm/>`
- `<PostList/>`
  - `<PostRow/>` for each row data

As parent component, `App` handles all actions related to form (`Add New/Reset`, `Save`, `Sync`) and row data (`Edit`, `Delete`).

```react
// assets/js/blog/App.js
import React, { useState } from 'react';
import { useDatabase } from '@nozbe/watermelondb/hooks'
import { v4 as uuidv4 } from 'uuid';

import syncData from './sync'
import PostList from './PostList'
import PostForm from './PostForm'

export default function App() {
    const database = useDatabase()
    const [post, setPost] = useState()
    const postsCollection = database.collections.get("posts");

    function clearPost() {
        setPost(undefined)
    }

    function onEdit(selectedPost) {
        setPost(selectedPost)
    }

    async function onDelete(selectedPost) {
        await database.action(async () => {
            await selectedPost.markAsDeleted()
        })
    }

    async function createPost(inputtedForm) {
        await database.action(async () => {
            const newPost = await postsCollection.create(post => {
                post._raw.id = uuidv4()
                post.title = inputtedForm.title
                post.content = inputtedForm.content
                post.likes = inputtedForm.likes
            });
        })
    }

    async function updatePost(currentPost, inputtedForm) {
        await database.action(async () => {
            await currentPost.update(post => {
                post.title = inputtedForm.title
                post.content = inputtedForm.content
                post.likes = inputtedForm.likes
            });
        })
    }

    return (
        <div>
            <PostForm post={post} clearPost={clearPost} createPost={createPost} updatePost={updatePost} syncData={() => syncData(database)} />
            <PostList onEdit={onEdit} onDelete={onDelete} />
        </div>
    )
}
```

## `PostForm.js`

```react
// assets/js/blog/PostForm.js
import React, { useState, useEffect } from 'react';

export default function PostForm({ post, clearPost, createPost, updatePost, syncData }) {
    const [title, setTitle] = useState(post ? post.title : "")
    const [content, setContent] = useState(post ? post.content : "")
    const [likes, setLikes] = useState(post ? post.likes : "")

    useEffect(() => {
        setTitle(post ? post.title : "")
        setContent(post ? post.content : "")
        setLikes(post ? post.likes : "")
    }, [post])

    const onReset = (e) => {
        e.preventDefault()
        clearForm()
    }

    const onSync = (e) => {
        e.preventDefault()
        syncData()
    }

    const onSubmit = async (e) => {
        e.preventDefault()

        const inputtedForm = { title, content, likes: parseInt(likes) }

        if (post) {
            await updatePost(post, inputtedForm)
        } else {
            await createPost(inputtedForm)
            clearForm()
        }
    }

    const clearForm = () => {
        setTitle("")
        setContent("")
        setLikes("")
        clearPost()
    }

    return (
        <form>
            <label>
                Title:
                <input type="text" value={title} onChange={(e) => setTitle(e.target.value)} />
            </label>
            <label>
                Content:
                <input type="text" value={content} onChange={(e) => setContent(e.target.value)} />
            </label>
            <label>
                Likes:
                <input type="number" value={likes} onChange={(e) => setLikes(e.target.value)} />
            </label>
            <button className="button button-outline" onClick={onReset}>Add New / Reset</button>
            <button onClick={onSubmit}>Save</button>
            <button className="button button-clear" onClick={onSync}>Sync</button>
        </form>
    )
}
```

## `PostList.js`

`PostList` is enhanced component wrapped with `withObservables` to become reactive whenever data in `posts` get added or deleted.

```react
// assets/js/blog/PostList.js
import React from 'react';
import withObservables from "@nozbe/with-observables";
import { withDatabase } from '@nozbe/watermelondb/DatabaseProvider'
import PostRow from './PostRow'

const PostList = ({ posts, onEdit, onDelete }) => (
    <table>
        <thead>
            <tr>
                <th>Title</th>
                <th>Content</th>
                <th>Likes</th>
                <th>Created At</th>
                <th>Updated At</th>
                <th>Actions</th>
            </tr>
        </thead>
        <tbody>
            {posts.map(post => <PostRow key={post._raw.id} post={post} onEdit={onEdit} onDelete={onDelete} />)}
        </tbody>
    </table>
)

export default withDatabase(withObservables([], ({ database }) => ({
    posts: database.collections.get('posts').query().observe(),
}))(PostList))
```

## `PostRow.js`

`PostRow` is also reactive component. It will automatically rerender the component whenever data changes (i.e. `post` get updated).

```react
// assets/js/blog/PostRow.js
import React from 'react';
import withObservables from "@nozbe/with-observables";

const PostRow = ({ post, onEdit, onDelete }) => (
    <tr key={post._raw.id}>
        <td>{post.title}</td>
        <td>{post.content}</td>
        <td>{post.likes}</td>
        <td>{post.createdAt.toString()}</td>
        <td>{post.updatedAt.toString()}</td>
        <td>
            <button onClick={(e) => {onEdit(post)}}>Edit</button>
            <button onClick={(e) => {onDelete(post)}}>Delete</button>
        </td>
    </tr>
)

export default withObservables(["post"], ({ post }) => ({
    post: post.observe()
}))(PostRow)
```

## `sync.js`

Along with the backend, this is the core of sync capability in our app.
On [the previous post]({{< ref "how-to-build-watermelondb-sync-backend-in-elixir.md#workaround-for-sync-on-client-side" >}}), I have explained in detail why it needs a workaround.
Please read that first.

```js
// assets/js/blog/sync.js
import { synchronize } from '@nozbe/watermelondb/sync'

export default async function syncData(database) {
    let latestVersionOfSession = 0
    let changesOfSession = {}

    await synchronize({
        database,
        pullChanges: async ({ lastPulledAt }) => {
            const response = await fetch(`http://localhost:4000/api/sync/pull?lastPulledVersion=${lastPulledAt || 0}`)
            if (!response.ok) {
                throw new Error(await response.text())
            }

            const { changes, latestVersion } = await response.json()
            latestVersionOfSession = latestVersion
            changesOfSession = changes

            return { changes, timestamp: latestVersion }
        },
        pushChanges: async ({ changes, lastPulledAt }) => {
            const response = await fetch(`http://localhost:4000/api/sync/push?lastPulledVersion=${lastPulledAt || 0}`, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify(changes)
            })
            if (!response.ok) {
                throw new Error(await response.text())
            }
            const { changes: changesFromPush, latestVersion } = await response.json()
            latestVersionOfSession = latestVersion
            changesOfSession = changesFromPush
        },
    })

    await synchronize({
        database,
        pullChanges: async ({ lastPulledAt }) => {
            return { changes: changesOfSession, timestamp: latestVersionOfSession }
        },
        pushChanges: async ({ changes, lastPulledAt }) => {
            throw new Error(await response.text())
        },
    })
}
```

## Run the Application

```shell
$ cd ../ # cd to project directory (blog_app)
$ mix phx.server
```

Open <http://localhost:4000>

## Final Source Code

All code (sync backend and frontend app) is available on <https://github.com/fahrinh/blog-labs/tree/master/2020-04-24/blog_app>