---
doc: DETAILS
package: EideticLib
repo: moot-semantics
authored_commit: 021ea704162f86fccea2f030ea9419dacc30a345
authored_date: 2026-07-04
sources:
  - path: Sources/EideticLib/EideticLib.swift
    blob: 1839588dc98279c1eee2786f1974457a5608742b
  - path: Sources/EideticLib/LatticeCodeState.swift
    blob: 7580261208b77c7e79363a510920c413054264a1
  - path: Sources/EideticLib/Segmenter.swift
    blob: c5313d93bf7ca1cbf577da6ffabc9197b5e788e2
---

# EideticLib Details

This document walks through every source file in the package. Read
`OVERVIEW.md` first for the big picture. The three files are
independent of one another, so they appear here in the order a new
reader would most likely meet them: the lookup surface, the code-state
classifier, and the sentence segmenter.

## EideticLib.swift

This file provides the module surface: the `EideticLib` enum, its
`lookup` functions, `classifyLatticeCode`, and the `Anchor` result
type.

`EideticLib` is a namespace enum, the same pattern LatticeLib uses for
its own module surface. It holds no reference data. The comment at the
top of the file states this explicitly: LatticeLib's FDC runtime owns
and caches the pinned lexicon, frame, and signatures; EideticLib parses
nothing of its own and simply calls through.

The two `lookup` overloads share one behavior: they never return a
fabricated answer. If `FDC.isAvailable` is false — meaning the FDC
artifacts bundled inside LatticeLib failed to load — both overloads
call `fatalError` and terminate the process. A comment in the code
ties this choice to a specific product mandate: "a sentinel identity
that persists IS a fabricated identity" (Bob's board item 7). The
distinction matters. A term that FDC genuinely cannot classify returns
UNRESOLVED — an empty code, not an error — because that is a true
statement about the term. A missing artifact bundle is not a fact
about any term; it is a broken build, and the correct response is to
stop immediately rather than let every caller silently receive
meaningless answers.

`version` is the module version string, `"0.1.0"`. Like LatticeLib's
version constant, it exists so a consumer can record which build of
EideticLib produced a given answer.

`classifyLatticeCode(_:knownCodes:)` sorts a candidate lattice code
string into one of `LatticeCodeState`'s three cases, using
`LatticeCodeGrammar.isWellFormed(_:)` for the shape check and the
caller-supplied `knownCodes` set for the known/pending distinction.
Passing the known-code set in as a parameter, rather than having
EideticLib own a canon, keeps EideticLib decoupled from any particular
caller's view of which codes currently resolve to a label. WHY this
matters: two callers can disagree about which codes are "known" (a
newly synced frame versus a stale one) without EideticLib taking a
side.

`lookup(_:)` is the default path. It guards on `FDC.isAvailable`, calls
`FDC.encodeAnchor(term)`, and converts the result to an `Anchor`. A
resolved code is reported at confidence 32 (medium), the fixed value
this library always emits, because FDC itself computes no calibrated
score. An UNRESOLVED term becomes an `Anchor` with an empty `code`, a
`nil` `wikidataQID`, and confidence 0 — the honest "nothing matched"
result, never a best-guess fallback.

`lookup(_:recordNovel:)` is the privacy variant. It produces the exact
same `Anchor` as `lookup(_:)` for the same term, but threads
`recordNovel` through to `FDC.encodeAnchor(_:recordNovel:)`. When
`false`, any novel token encountered while building the term's concept
bag is not written into LatticeLib's shared learning pool. WHY this
exists: the GLK (memory-capture) intake path classifies user-supplied
memory content before deciding whether to store it, and that content
must never leak a single token into a shared, cross-session pool —
even a rejected or sensitive capture must leak nothing, because
classification happens before the write decision, not after.

`Anchor` is a plain `Equatable`, `Sendable`, `Codable` struct with four
fields: `code` (empty string means UNRESOLVED), `wikidataQID`
(optional), `confidence` (a `UInt8` drawn from the fixed set 0/16/32/48/56
— null/low/medium/high/verified — of which EideticLib currently only
ever emits 0 or 32), and `dataVersion` (the FDC signatures version that
produced the answer, letting a caller record provenance). The struct's
doc comment notes it is byte-identical to the Rust port's `Anchor`
under JSON encoding, which is what lets a Swift app and a Rust daemon
exchange anchors through a shared store.

## LatticeCodeState.swift

This file provides `LatticeCodeState`, the three-state classification
of a candidate lattice code, and `LatticeCodeGrammar`, the grammar
check behind it.

The file's opening comment frames the problem directly: a lattice code
shown to EideticLib is either malformed (fails the grammar), known
(well-formed and present in the caller's current canon), or pending
(well-formed, but not yet in the caller's canon). WHY a third state is
needed at all: FDC frames are versioned and ship new codes over time,
and MOOTx01 estates can federate, meaning one estate's stored code may
predate another estate's local frame. Rejecting an unrecognized but
well-formed code would lose data permanently; a pending code instead
round-trips intact through storage until a newer frame resolves it.

`LatticeCodeState` conforms to `Sendable`, `Hashable`, and `Codable`.
The `Codable` conformance is not incidental — it is, per the file's own
comment, "the core invariant of the launch plan's valid-but-unknown-code
requirement": a pending code must survive being written to and read
back from storage without alteration. `rawCode` returns the original
string regardless of which case matched, so a storage layer can
persist the raw value without unpacking the enum first. `isWellFormed`
is `true` for `.known` and `.pending` and `false` only for
`.malformed`, which is useful in tests asserting that pending codes are
still accepted, just not yet resolved.

`LatticeCodeGrammar` is a second, independent implementation of the
same grammar LatticeLib's `Code.isWellFormed(_:)` enforces: three ASCII
digits, optionally followed by a dot and one to eight further ASCII
digits. WHY it is reimplemented here rather than imported from
LatticeLib: the file's comment explains that EideticLib can validate a
code's shape without loading LatticeLib's frame, lexicon, and
signatures at all — a caller who only wants to know "is this string
shaped like a code" pays none of the cost of the classification engine.
The two implementations are kept in agreement by shared conformance
tests, not by shared code.

`maxExtensionDigits` is a pinned constant, `8`, matching LatticeLib's
own `Code.maxExtensionDigits`. `isWellFormed(_:)` splits the string on
its first `.`, requires the integer part to be exactly three ASCII
digits, and — if a decimal part is present — requires it to be one to
eight ASCII digits with no other characters. An empty decimal part
(a trailing dot with nothing after it) fails the check.

## Segmenter.swift

This file provides `sentences(_:)` and `sentencesByDelimiter(_:)`, both
declared in an extension on the `EideticLib` enum.

The file's opening comment draws an explicit parallel to LatticeLib's
`WordClassTagger`: that file splits word tagging into a deterministic
reference path (the fast-path table plus HMM) and an accelerated,
Apple-only path (`NLTagger`, opt-in). `Segmenter` applies the same
shape to sentence splitting instead of word tagging.
`sentencesByDelimiter(_:)` is the deterministic reference,
identical on every platform. `sentences(_:)` is the platform-routed
entry point most callers should use: on Apple platforms it defers to
`NLTokenizer`, which correctly handles abbreviations ("Dr."), locale-aware
punctuation, and other edge cases a naive delimiter split gets wrong; on
non-Apple platforms it falls back to `sentencesByDelimiter`.

WHY platform divergence here is acceptable, when LatticeLib treats
platform divergence in word tagging as unacceptable: the comment cites
the "apple-nlp-accel constitutional constraint (C-2)." Downstream
consumers of sentence segments key each chunk by `(sourceID,
startOffset, text)` — a content address — rather than by chunk index.
Two devices producing slightly different sentence boundaries for the
same document therefore produce a superset of chunks under an
append-only conflict policy, not a pair of conflicting writes that need
reconciling. This is a materially weaker consistency requirement than
FDC's lattice codes, which must be byte-identical across devices for
federated recall to work at all.

Both functions guarantee total coverage: for non-empty input, every
character appears in exactly one output segment, and joining the
segments reproduces the original text. This holds even in pathological
cases — text that is nothing but delimiters, or text with no delimiter
at all, which returns the entire input as a single segment. Empty
input returns an empty array from both functions. The file's history
comment notes the code was relocated here in 2026-05-27 (a change
tagged F16) from `CorpusKit/Sources/CorpusKit/Chunker.swift`, to
centralize every linguistic-pipeline stage — tokenizing, normalizing,
stemming, tagging, and now segmenting — behind LatticeLib and
EideticLib rather than scattering pieces of the pipeline across
consuming kits.

`sentences(_:)` builds an `NLTokenizer(unit: .sentence)`, enumerates
sentence-unit ranges over the input, and collects the corresponding
substrings. If the tokenizer produces no segments for non-empty input,
the function falls back to returning the whole input as one segment,
preserving the total-coverage guarantee even in that edge case.

`sentencesByDelimiter(_:)` walks the string once, splitting after every
`.`, `!`, `?`, or `\n`, and keeping the terminator attached to the
segment that ends there. Any trailing text after the final terminator
becomes its own segment. This is the function the Rust port's
`segmenter::sentences` mirrors exactly, because Rust has no
NLTokenizer-equivalent acceleration to port.

## Rust Port and Conformance

The `rust/` directory holds the second leg: `lib.rs` (the crate surface,
re-exporting `lookup` and `classify`), `anchor.rs` (`Anchor`, with an
explicit `#[serde(rename = "wikidataQID")]` so the JSON key matches
Swift's `Codable` output exactly rather than the auto-derived
`wikidataQid`), `lattice_code_state.rs` (`LatticeCodeState`,
`LatticeCodeGrammar`, and `classify_lattice_code`, wire-tagged as
`{"state": ..., "code": ...}` to match Swift's encoding), and
`segmenter.rs` (the delimiter algorithm only — there is no Rust
counterpart to the Apple-only `NLTokenizer` path, by design).

The Rust crate depends on the sibling `lattice-lib` crate for
`Fdc::encode_anchor` and `Fdc::is_available`, the same relationship
Swift's `EideticLib` target has to the LatticeLib Swift package.
`rust::lookup` panics under the identical condition Swift's `lookup`
treats as a `fatalError`: unavailable FDC artifacts.

Cross-language conformance runs on two fixtures.
`Tests/SharedVectors/lookup_vectors.json` (schema version 2, 26 vectors)
pins expected FDC codes and Wikidata Q-IDs for a set of inputs,
including edge cases such as punctuation-only text; both
`rust/tests/lookup_conformance_test.rs` and the Swift
`LookupConformanceTests.swift` load the same file and must match it
exactly. `Tests/SharedVectors/word_class_vectors.json` is a second
fixture, shared with LatticeLib's own word-classification conformance
surface, exercised here because the EideticLib test target links
LatticeLib directly. When you change `lookup`, `classifyLatticeCode`,
or either segmentation function, update both legs and rerun both test
suites; the fixtures are the contract.
