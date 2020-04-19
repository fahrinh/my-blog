---
title: "Generate Sequential UUIDv4 in Elixir"
date: 2020-04-19T12:38:13+07:00
categories: ["Elixir"]
tags: ["UUIDv4"]
draft: true
toc: true
---

The idea is simple. This sequential UUIDv4 (128 bit) consists of `unix timestamp (32 bit) + random bytes (96 bit)`

<!--more-->

## UUIDv4

Basic form of normal UUIDv4 (128 bit) in hex numbers (32 digits) [^1] :

```text
xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx
```

- Each digit is 4 bit.
- `x` is any hexadecimal digit
- `y` is either `8`, `9`, `A`, or `B`
- `4` and `y` are version flags indicate version 4 UUID [^2]
    - `y` is 4 bit: `10zz` where `z` is random bit. Hence, the possible value of `y` is `1000` (`8`), `1001` (`9`), `1010` (`A`), `1011` (`B`)

[^1]: https://stackoverflow.com/questions/19989481/how-to-determine-if-a-string-is-a-valid-v4-uuid/19989922#19989922
[^2]: http://tools.ietf.org/html/rfc4122#section-4.4