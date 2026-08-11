# JokeBase sequence 19 demo video

**Video:** `assets/jokebase-sequence-19-live-demo.mp4`

**Format:** 29-second silent vertical MP4, 1080 × 1350 pixels, H.264 video.

## What it demonstrates

The clip is based on a fresh live execution of the promoted `JokeBase-v1.wasm` artifact:

1. create a bounded unique TEXT table;
2. insert `coffee` and `tea`;
3. query membership for `coffee`;
4. save a host-managed snapshot;
5. insert `after-save` and confirm it exists;
6. restore the snapshot;
7. confirm `coffee` remains and `after-save` no longer exists;
8. show the honest limitation that `SELECT * FROM t7` is unsupported.

The live artifact was 18,438 bytes with SHA-256 `ca3c9d4d3c0a76767281410e1df5da713f1e5f40e26c5635b3106eb09458580b`.

## Authenticity note

The contents come from a live black-box run of the public artifact through its documented SQL and snapshot interface. The terminal applications available in this environment could not be directly controlled for a literal operating-system screen capture, so the resulting video is a terminal-style rendering of that exact captured output. It is not a recording of a graphical product interface, and it makes no claim to be one.

This distinction is deliberate: the video shows what JokeBase actually did, while avoiding an edited or fictional result.

## Suggested LinkedIn caption

> A tiny live demo of JokeBase: the database built without anyone ever reading or writing its implementation source code.
>
> In 29 seconds: create a table, insert values, query it, save state, add a value, restore the old state, and watch the later value disappear.
>
> Then the important bit: `SELECT *` fails. JokeBase is real, but it is not a general SQL database—and the limits are part of the experiment.
>
> The interesting question is not “can an AI emit bytes?” It is whether an opaque artifact can become trustworthy through tests, evidence, hashes, and honest boundaries.
>
> https://github.com/bitkojine/JokeBase
