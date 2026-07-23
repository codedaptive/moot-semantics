---
doc: DETAILS
package: EideticLib
repo: moot-semantics
authored_commit: 4b160824be9616aafb464187f74e884b1ff6d61d
authored_date: 2026-07-23
sources:
  - path: Sources/EideticLib/EideticLib.swift
    blob: 545be504130a9becae7cbe6e7d58a243ccb07a64
  - path: Sources/EideticLib/LatticeCodeState.swift
    blob: 7580261208b77c7e79363a510920c413054264a1
  - path: Sources/EideticLib/Segmenter.swift
    blob: c5313d93bf7ca1cbf577da6ffabc9197b5e788e2
---

# EideticLib Details

## Current Release Details

`EideticContentKind` adds text and code choices.
The content-aware lookup maps code to FDC `005`.
It adds a language Q-ID only when local detection has one clear winner.
The caller still controls novel-token recording.

This document walks through every source file in the package. Read
`OVERVIEW.md` first for the big picture. The three files stand
independent of one another. They appear here in the order a new reader
would most likely meet them: the lookup surface, the code-state
classifier, and the sentence segmenter.

## EideticLib.swift

This file provides the module surface. That surface is the
`EideticLib` enum, its `lookup` functions, `EideticContentKind`,
`classifyLatticeCode`, and the `Anchor` result type.

`EideticLib` is a namespace enum, the same pattern LatticeLib uses for
its own module surface. It holds no reference data. A comment at the
top of the file states this plainly. LatticeLib's FDC runtime owns and
caches the pinned lexicon, frame, and signatures. EideticLib parses
nothing of its own. It simply calls through.

The lookup forms share one behavior. They never return a
made-up answer. Suppose `FDC.isAvailable` is false, meaning the FDC
artifacts bundled inside LatticeLib failed to load. Then both overloads
call `fatalError` and stop the process. A comment in the code names a
specific product mandate for this choice. It quotes Bob's board item
seven: "a sentinel identity that persists IS a fabricated identity."
The distinction matters. A term that FDC truly cannot classify returns
UNRESOLVED. This is an empty code, not an error. That empty code is a
true statement about the term. A missing artifact bundle is different. It
is not a fact about any term. It is a broken build. The correct
response is to stop at once, rather than let every caller silently
receive a meaningless answer.

The content-aware form accepts `EideticContentKind`. Text uses the
normal FDC classifier. Nonempty code uses FDC `005`. A clear local
language match also supplies its pinned Wikidata Q-ID. The path does
not use a network or a model service.

`version` is the module version string, `"0.1.0"`. Like LatticeLib's
own version constant, it serves one purpose. A consumer can record
which build of EideticLib produced a given answer.

`classifyLatticeCode(_:knownCodes:)` sorts a candidate lattice code
string into one of `LatticeCodeState`'s three cases. It uses
`LatticeCodeGrammar.isWellFormed(_:)` for the shape check. It uses the
caller-supplied `knownCodes` set for the known-versus-pending
distinction. The known-code set arrives as a parameter, rather than
living inside EideticLib as an owned canon. This keeps EideticLib
decoupled from any single caller's view of which codes currently
resolve to a label. Here is why that matters. Two callers can disagree
about which codes count as known. One might hold a newly synced frame.
The other might hold a stale one. EideticLib does not take a side.

`lookup(_:)` is the default path. It guards on `FDC.isAvailable`. It
calls `FDC.encodeAnchor(term)`. Then it converts the result to an
`Anchor`. A resolved code is reported at a fixed confidence of 32, or
medium. FDC computes no calibrated score of its own, so EideticLib
always emits this same value. An UNRESOLVED term becomes an `Anchor`
with an empty `code`, a `nil` `wikidataQID`, and confidence 0. That is
the honest "nothing matched" result, never a best-guess fallback.

`lookup(_:recordNovel:)` is the privacy variant. It produces the exact
same `Anchor` as `lookup(_:)` for the same term. It also threads
`recordNovel` through to `FDC.encodeAnchor(_:recordNovel:)`. When that
flag is `false`, any novel token found while building the term's
concept bag stays out of LatticeLib's shared learning pool. Here is why
this variant exists. The GLK memory-capture intake path classifies
user-supplied memory content. It does this before it decides whether
to store that content. That content must never leak even a single
token into a shared, cross-session pool. This holds even for a
rejected or sensitive capture. Classification runs before the write
decision, not after.

`Anchor` is a plain `Equatable`, `Sendable`, `Codable` struct with four
fields. `code` is empty when the term is UNRESOLVED. `wikidataQID` is
optional. `confidence` is a `UInt8`. It is drawn from a fixed set of
five values: 0, 16, 32, 48, and 56. These stand for null, low, medium,
high, and verified. EideticLib currently only ever emits 0 or 32.
`dataVersion` is the FDC signatures version that produced the answer.
This lets a caller record provenance. The struct's doc comment notes
that it is byte-identical to the Rust port's `Anchor` under JSON
encoding. That identity is what lets a Swift app and a Rust daemon
exchange anchors through a shared store.

## LatticeCodeState.swift

This file provides two things: `LatticeCodeState` and
`LatticeCodeGrammar`. `LatticeCodeState` is the three-state
classification of a candidate lattice code. `LatticeCodeGrammar` is the
grammar check behind it.

The file's opening comment frames the problem directly. A lattice code
shown to EideticLib falls into one of three states. A malformed code
fails the grammar. A known code is well formed and present in the
caller's current canon. A pending code is well formed, but not yet
present in that canon. Here is why a third state exists at all. FDC
frames carry a version and ship new codes over time. MOOTx01 estates
can federate, so one estate's stored code may predate another estate's
local frame. Rejecting an unrecognized but well-formed code would lose
data for good. A pending code instead round-trips intact through
storage until a newer frame resolves it.

`LatticeCodeState` conforms to `Sendable`, `Hashable`, and `Codable`.
The `Codable` conformance is not incidental. The file's own comment
calls it "the core invariant of the launch plan's
valid-but-unknown-code requirement": a pending code must survive being
written to storage and read back without alteration. `rawCode` returns
the original string regardless of which case matched, so a storage
layer can persist the raw value without unpacking the enum first.
`isWellFormed` reads `true` for `.known` and `.pending`, and `false`
only for `.malformed`. Tests rely on this to assert that pending codes
are still accepted, just not yet resolved.

`LatticeCodeGrammar` is a second, separate check. It enforces the same
grammar as LatticeLib's `Code.isWellFormed(_:)`. That grammar requires
three ASCII digits. It allows an optional dot, followed by one to eight
more ASCII digits.

Here is why the grammar is reimplemented, rather than imported from
LatticeLib. The file's comment gives the reason. EideticLib can check a
code's shape without loading LatticeLib's frame, lexicon, and
signatures at all. A caller who only wants to know whether a string is
shaped like a code pays none of the cost of the full classification
engine. Shared conformance tests keep the two implementations in
agreement. The implementations do not share code.

`maxExtensionDigits` is a pinned constant, `8`, matching LatticeLib's
own `Code.maxExtensionDigits`. `isWellFormed(_:)` splits the string on
its first `.`. It requires the integer part to be exactly three ASCII
digits. If a decimal part is present, it requires that part to hold one
to eight ASCII digits with no other characters. An empty decimal part,
meaning a trailing dot with nothing after it, fails the check.

## Segmenter.swift

This file provides `sentences(_:)` and `sentencesByDelimiter(_:)`, both
declared in an extension on the `EideticLib` enum.

The file's opening comment draws a direct parallel to LatticeLib's
`WordClassTagger`. That file splits word tagging into two paths. One
path is deterministic: the fast-path table plus an HMM. The other path
is accelerated and Apple-only, built on `NLTagger`. Callers must opt
into the accelerated path. `Segmenter` applies that same shape to
sentence splitting instead of word tagging. `sentencesByDelimiter(_:)`
is the deterministic reference. It behaves the same on every platform.
`sentences(_:)` is the platform-routed entry point most callers should
use. On Apple platforms it defers to `NLTokenizer`. This tool correctly
handles tricky abbreviations. For example, it does not split "Dr."
into two sentences. It also handles locale-aware punctuation and other
edge cases that a naive delimiter split gets wrong. On non-Apple
platforms it falls back to `sentencesByDelimiter`.

Why is this divergence acceptable here? LatticeLib treats the same
kind of divergence as unacceptable in word tagging. The comment cites
the "apple-nlp-accel constitutional constraint (C-2)." Downstream
consumers of sentence segments key each chunk by a content address.
That address is `(sourceID, startOffset, text)`, not a chunk index.
Two devices might split the same document into slightly different
sentences. Under an append-only conflict policy, this produces a
superset of chunks. It does not create a conflict that needs
reconciling. This is a much weaker consistency rule than the one FDC's
lattice codes carry. Those codes must match byte for byte across
devices, or federated recall fails.

Both functions guarantee total coverage. For non-empty input, every
character appears in exactly one output segment. Joining the segments
back together reproduces the original text. This holds even in
pathological cases. Consider text that is nothing but delimiters.
Consider also text with no delimiter at all. That case returns as a
single segment holding the whole input. Empty input returns an empty
array from both functions. The file's history comment notes that the
code moved here on 2026-05-27, in a change tagged F16, from
`CorpusKit/Sources/CorpusKit/Chunker.swift`. That move gathered every
text-pipeline stage under LatticeLib and EideticLib. Those stages are
tokenizing, normalizing, stemming, tagging, and now segmenting. The
move ended the old pattern of scattering pipeline pieces across the
kits that consume them.

`sentences(_:)` builds an `NLTokenizer(unit: .sentence)`, enumerates
sentence-unit ranges over the input, and collects the corresponding
substrings. Suppose the tokenizer produces no segments for non-empty
input. Then the function falls back to returning the whole input as one
segment, which preserves the total-coverage guarantee even in that edge
case.

`sentencesByDelimiter(_:)` walks the string once. It splits the text
after each terminator: a period, an exclamation point, a question
mark, or a newline. It keeps each terminator attached to the segment
that ends there. Any trailing text after the final terminator becomes
its own segment. This is the function the Rust port's
`segmenter::sentences` mirrors exactly, because Rust has no
NLTokenizer-equivalent acceleration to port.

## Rust Port and Conformance

The `rust/` directory holds the second leg. `lib.rs` is the crate
surface, re-exporting `lookup` and `classify`. `anchor.rs` defines
`Anchor`, with an explicit `#[serde(rename = "wikidataQID")]` so the
JSON key matches Swift's `Codable` output exactly, rather than the
auto-derived `wikidataQid`. `lattice_code_state.rs` defines three
items: `LatticeCodeState`, `LatticeCodeGrammar`, and
`classify_lattice_code`. The wire format is tagged as
`{"state": ..., "code": ...}`, matching Swift's encoding.
`segmenter.rs` implements the delimiter algorithm only. There is no
Rust counterpart to the Apple-only `NLTokenizer` path, by design.

The Rust crate depends on the sibling `lattice-lib` crate for
`Fdc::encode_anchor` and `Fdc::is_available`. This is the same
relationship Swift's `EideticLib` target holds with the LatticeLib
Swift package. `rust::lookup` panics under the identical condition that
Swift's `lookup` treats as a `fatalError`: unavailable FDC artifacts.

Cross-language conformance runs on two fixtures.
`Tests/SharedVectors/lookup_vectors.json`, schema version 2, holds
twenty-six vectors. It pins expected FDC codes and Wikidata Q-IDs for a
set of inputs, including edge cases such as punctuation-only text. Both
`rust/tests/lookup_conformance_test.rs` and the Swift
`LookupConformanceTests.swift` load that same file and must match it
exactly. `Tests/SharedVectors/word_class_vectors.json` is a second
fixture. LatticeLib's own word-classification conformance surface
shares this fixture too. The EideticLib test target links LatticeLib
directly, so this fixture applies here as well. Suppose you change
`lookup`, `classifyLatticeCode`,
or either segmentation function. Then update both legs, and rerun both
test suites. The fixtures are the contract.
