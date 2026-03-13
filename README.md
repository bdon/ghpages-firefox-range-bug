## Status

**As of March 13, 2026, this issue seems to have been resolved.**

---


This is a demonstration of how recent changes to GitHub Pages makes it impossible to use HTTP Ranges with Firefox.

The page hosted at [bdon.github.io/ghpages-firefox-range-bug/](https://bdon.github.io/ghpages-firefox-range-bug/) works correctly on Chrome and Safari, but will throw a 416 error on Firefox.

## Explanation

The file `object_short` is the text `Hello World!` compressed with gzip. 

The page at `index.html` fetches all bytes using a range request `Range: bytes=0-38` and then uses the `DecompressionStream` API to get the original contents.

The file `object_long` is the same content as `object_short` but with 5 megabytes (1024 * 1024 * 5, or 5242880 bytes) of zeroes prepended:

```sh
( dd if=/dev/zero bs=1M count=5 && cat object_short ) > object_long
```

When we instead request the same byte range at a 5MB offset, `bytes=5242880-5242918`, the GitHub pages server returns a `416 Range Not Satisfiable` error.

## Hypothesis

If you look at Firefox's request headers, it is the only one to send the `Accept-Encoding
  gzip, deflate, br, zstd, identity` header in the request. Other browsers send only `identity`.

In GitHub's CDN configuration, responses are being force-compressed if certain conditions are met:

1. The request includes the `Accept-Encoding: gzip` header
2. The resource is over a certain size (5MB?)

You can see this because the 416 response does return a `content-range
  bytes */5183` header, indicating the CDN thinks the resource is about 5kB, which is similar to the result of `gzip object_long`.
  
## Workaround

Renaming `.pmtiles` to `.gz` or `.pmtiles.gz` should disable server-side compression completely.

A [PR is open](https://github.com/jshttp/mime-db/pull/398) on the `mime-db` project to mark `.pmtiles` as uncompressible. 
