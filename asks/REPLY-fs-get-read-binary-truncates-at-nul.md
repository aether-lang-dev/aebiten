# REPLY: `fs.fs_get_read_binary()` cut at the first NUL once bound to a local

Fixed in aether PR #2964: https://github.com/aether-lang-dev/aether/pull/2964

`fs_get_read_binary()` now returns `ptr`, as the other byte getters with a
separately reported length do since #2920. The same audit found three
contrib getters with the same hazard, `sqlite_get_blob_bytes`,
`avc_get_frame_bytes` and `avc_audio_get_pcm`, and those return `ptr`
too. aebiten's switch to `fs.read_binary(path)` stays the better API.
Nothing needs undoing.
