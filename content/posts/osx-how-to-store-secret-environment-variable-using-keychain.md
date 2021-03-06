---
title: "OSX: How to Store Secret Environment Variable using Keychain"
date: 2019-10-06T08:26:27+07:00
draft: true
---

As a software developer, we always deal with configuration containing credentials such as database password, third-party API key/token, etc.
If we embrace [12 Factor](https://www.12factor.net) methodology, the configuration is stored in environment variables.
This is an example how to run application along with its configuration in environment variables:

```shell
$ DB_PWD=S3cr37pass REDIS_PWD=P455 ./my-app
```

For deployment on servers, secret credentials are usually managed by dedicated configuration service like [Vault](https://www.vaultproject.io) or 

<!--more-->

**IF NOT, YOU HAVE TO WORRY! DON'T STORE CREDENTIALS IN PLAIN TEXT!**
