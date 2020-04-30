---
title: "Building an Offline First React Web App Using WatermelonDB in Phoenix (Elixir)"
date: 2020-04-29T10:58:39+07:00
categories: ["React"]
tags: ["Phoenix", "WatermelonDB", "Offline First App"]
draft: true
---

We will build a web application that supports local db synced with remote db. This post is a continuation from the previous post: [How to Build WatermelonDB Sync Backend in Elixir]({{< ref "how-to-build-watermelondb-sync-backend-in-elixir.md" >}}).

<!--more-->

## Installing Dependencies

```shell
$ cd assets
$ npm i react react-dom @nozbe/watermelondb @nozbe/with-observables
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

For now, it only displays "Hello World". We will code it later.

```react
// assets/js/blog/index.js
import React from 'react';
import { render } from 'react-dom';

const rootElement = document.getElementById('blog-app');

render(<div> Hello World! </div>, rootElement);
```

## Setup `Database`

```react
// assets/js/blog/index.js

// ...
import { Database } from '@nozbe/watermelondb'
import LokiJSAdapter from '@nozbe/watermelondb/adapters/lokijs'

import schema from './model/schema'
import Post from './model/Post'

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
// ...
```

### Schema

```react
// assets/js/blog/model/schema.js
import { appSchema, tableSchema } from '@nozbe/watermelondb'

export const mySchema = appSchema({
    version: 1,
    tables: [
        tableSchema({
            name: 'posts',
            columns: [
                { name: 'title', type: 'string' },
                { name: 'content', type: 'string' },
                { name: 'likes', type: 'number', isOptional: true },
                { name: 'created_at', type: 'number' },
                { name: 'updated_at', type: 'number' },
            ]
        })
    ]
})
```

### Model

```react
// assets/js/blog/model/Post.js
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

## App

```react
// assets/js/blog/index.js
// ...
import App from './App'
// ...
const rootElement = document.getElementById('blog-app');
render(<App db={database} />, rootElement);
```

