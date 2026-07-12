# kotoba-lang/org-w3-webcodecs

Raw **W3C WebCodecs** JS API — a thin binding layer, one function per spec
call, for `VideoDecoder`/`VideoEncoder`/`AudioDecoder`/`AudioEncoder` and
their chunk/frame types. Structured after `kotoba-lang/org-w3-webgpu`
(ADR-2607051400) — the video-domain (`eizo`) analog of that repo's role for
3D, per **ADR-2607121400**
(`90-docs/adr/2607121400-kami-ongaku-eizo-commercial-grade-cljs-stack.md`,
`com-junkawasaki/root`).

## Why this repo exists

Same split pattern as `org-w3-webgpu`/`org-materialx`/`org-openusd`: an
external standard's spec gets its own narrow-scope binding repo, and the
kotoba-lang-specific EDN domain vocabulary that consumes it lives one level
up. Here, `kotoba-lang/utsushi` (EDN container/filtergraph/codec layer) is
that consumer — this repo has **zero H.264/AAC/Opus bitstream logic**
(that's `utsushi`'s job); it only exposes the browser's `VideoDecoder`/
`VideoEncoder`/etc. constructors and methods cleanly.

**Scope is deliberately narrow: 1:1 spec bindings only.** Config objects
(`VideoDecoderConfig`, `VideoEncoderConfig`, ...) are plain JS objects with
the spec's own camelCase keys, built by the caller — this wrapper does not
translate Clojure maps into configs, for the same reason `org-w3-webgpu`
doesn't (that translation layer belongs one level up, in `utsushi`).

## One difference from org-w3-webgpu: a portable sub-namespace

Unlike WebGPU, WebCodecs has one piece of logic with **no browser
dependency**: parsing/constructing the `codec` config field's string value
(e.g. `"avc1.42001f"` for AVC/H.264 — profile_idc + constraint_set flags +
level_idc, per the WebCodecs AVC Codec Registration spec / RFC 6381). That
logic lives in `w3.webcodecs.codec-string` (`.cljc`, not `.cljs`) and has a
real `clojure.test`/`cljs.test` suite — see `test/`. Everything else
(`w3.webcodecs`, the actual `VideoDecoder`/`VideoEncoder` binding) stays
`.cljs`-only and untested here, same as `org-w3-webgpu`: there is no JVM or
Node WebCodecs implementation, so correctness for that half is pinned by
`utsushi`'s own browser E2E tests once it wires through this repo.

## API

- `w3.webcodecs` — `VideoDecoder`/`VideoEncoder`/`AudioDecoder`/
  `AudioEncoder` construction, `configure!`, `decode!`/`encode!`,
  `EncodedVideoChunk`/`EncodedAudioChunk`/`VideoFrame`/`AudioData`
  construction, `flush!`/`close!`, and the `isConfigSupported()` capability
  checks for all four codec types (required for capability-gated fallback —
  see ADR-2607121400's completion gates: never assume support, never
  silently degrade).
- `w3.webcodecs.codec-string` — `parse-avc-codec-string`/
  `format-avc-codec-string` (round-trips profile/constraint-flags/level,
  including the level-1b/level-1.1 ambiguity at `level_idc == 11`, which
  depends on `constraint_set3_flag`), plus a pass-through
  `parse-simple-codec` for fixed-token codecs like `"opus"`.

```clj
(require '[w3.webcodecs :as wc]
         '[w3.webcodecs.codec-string :as cs])

(-> (wc/video-decoder-config-supported!
      #js {:codec (cs/format-avc-codec-string {:profile :baseline :level 3.1})
           :codedWidth 1280 :codedHeight 720})
    (.then (fn [^js support]
             (when (.-supported support)
               (let [dec (wc/make-video-decoder
                           #js {:output (fn [frame] (wc/close-video-frame! frame))
                                :error  (fn [e] (js/console.error e))})]
                 (wc/configure-video-decoder! dec (.-config support)))))))
```

## Status

**v0 — thin binding layer + portable codec-string module.** No codec
bitstream implementation (that's `utsushi`'s H.264 encode/decode work,
landing alongside this repo). No real-browser E2E yet — per ADR-2607121400's
completion gates, real-browser WebCodecs decode/encode round-trip
verification is a separate, later completion item, not claimed here.

## Develop

```
clojure -M:test   # w3.webcodecs.codec-string only (portable, no browser needed)
clojure -M:lint
```

`w3.webcodecs` itself (the `VideoDecoder`/`VideoEncoder` binding) has no
runtime test suite here, same reasoning as `org-w3-webgpu`: `VideoDecoder`/
`VideoEncoder` don't exist under JVM/Node, so there's nothing to construct
outside a real browser. A cljs compile-check (see CI) catches
syntax/require errors in that half.
