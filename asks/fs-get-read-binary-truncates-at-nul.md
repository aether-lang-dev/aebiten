# `fs.fs_get_read_binary()` is cut at the first NUL once bound to a local (Aether 0.818.0 to 0.824.0)

Found bringing aebiten to 0.823.0: `png.png_load` returned null for every
PNG, because the file bytes it read stopped after the 8-byte signature.

## Repro

```aether
import std.fs
import std.string
import std.bytes
main() {
    p = "tests/testdata/text.png"          // any file with a NUL byte
    ok = fs.fs_try_read_binary(p)
    n = fs.fs_get_read_binary_length()
    s = fs.fs_get_read_binary()
    db = bytes.new(n)
    _ = bytes.copy_from_string(db, 0, s, n)
    println("n=${n} byte12=${bytes.get(db, 12)}")
}
```

0.817.0 prints `n=3743 byte12=73`, and 0.823.0 and 0.824.0 print `n=3743 byte12=0`.

## Why

std.fs still declares the split accessor as a C string:

```aether
extern fs_get_read_binary() -> string
```

Since 0.818's R1, a string local owns what it holds, so `s = fs_get_read_binary()`
copies the borrowed buffer through `aether_uniform_heap_str`, which is strlen
based. Only the bytes up to the first NUL survive. 0.818's notes say the other
byte getters were changed to return `ptr` ("bytes whose length C reports
separately are not a C string", #2920), for example `zlib_get_inflate_bytes`.
`fs_get_read_binary` was missed, and std.fs still exports it.

## Ask

Declare `fs_get_read_binary() -> ptr`, as was done for the zlib, crypto and
read getters. Alternatively, stop exporting it in favour of
`fs.read_binary(path)`, which is length-aware.

## What aebiten does meanwhile

`png_load` reads with `fs.read_binary(path)`, the tuple form, which copies
with `string_new_with_length`. That is the better API anyway, so this is not a
workaround to undo later.
