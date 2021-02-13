---
title: "Generate Sequential UUIDv4 in Elixir"
date: 2020-04-19T12:38:13+07:00
categories: ["Elixir"]
tags: ["UUIDv4"]
draft: false
toc: true
---

The idea is simple. Sequential UUIDv4 (128 bit) consists of `unix timestamp (32 bit) + random bytes (96 bit)`

<!--more-->

## UUIDv4

Basic form of normal UUIDv4 (128 bit) in hex numbers (32 digits) [^1] :

```text
xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx
```

- Each digit is 4 bit.
- `x` is any random hexadecimal digit
- `y` is either `8`, `9`, `A`, or `B`
- `4` and `y` are version flags indicate version 4 UUID [^2]
  - `y` is 4 bit: `10zz` where `z` is random bit. Hence, the possible value of `y` is `1000` (`8`), `1001` (`9`), `1010` (`A`), `1011` (`B`)

## Sequential UUIDv4

A bit modification on normal form where first 32 bits are UNIX timestamp (or 8 digit in hex number) and remaining bits comply with UUIDv4 spec as explained above.

```text
tttttttt-xxxx-4xxx-yxxx-xxxxxxxxxxxx
```

- `t` is UNIX timestamp in hex digit

Or in binary representation, the 128-bit sequential UUIDv4 consists of five segments:

```text
[unixtime:32][random:16]['4':4][random:12]['2'(first two bits of `y`):2][random:62]
```

### Implementation in Elixir

Thanks to binary pattern matching. The implementation in Elixir is trivial.

```elixir
defmodule SeqUUIDv4 do
  def generate() do
    unix_time = DateTime.utc_now() |> DateTime.to_unix()

    <<_r0::32, r1::16, _r2::4, r3::12, _r4::2, r5::62>> = :crypto.strong_rand_bytes(16)
    <<unix_time::32, r1::16, 4::4, r3::12, 2::2, r5::62>>
  end
end
```

The output of `generate/0` is binary.
To encode as hex numbers use `Base.encode16/1`:

```elixir
SeqUUIDv4.generate() |> Base.encode16()
```

[^1]: https://stackoverflow.com/questions/19989481/how-to-determine-if-a-string-is-a-valid-v4-uuid/19989922#19989922
[^2]: http://tools.ietf.org/html/rfc4122#section-4.4