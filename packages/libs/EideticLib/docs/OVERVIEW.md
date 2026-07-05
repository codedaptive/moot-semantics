---
doc: OVERVIEW
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

# EideticLib Overview

## What This Library Does

EideticLib sits in front of LatticeLib's classification engine. It gives
callers one simple way to classify a term. A caller passes in a term.
EideticLib returns an `Anchor`.

An `Anchor` holds four fields: a lattice code, a Wikidata Q-ID, a
confidence value, and a data version. A Wikidata Q-ID is a public
identifier for one concept. It works the same way across every
language. For example, `Q144` names the concept "dog."

EideticLib holds no reference data of its own. Every lookup calls
`FDC.encodeAnchor`. That call is the entry point into LatticeLib's
Frame-Directed Classification engine, known as FDC.

EideticLib also carries two small utilities. The first checks whether
a lattice code string is well formed and known. The second splits text
into sentences. Neither utility needs the FDC engine to run. All three
surfaces share one goal: give callers a clear answer without asking
them to learn LatticeLib's internals.

## The Problem It Solves

LatticeLib's `FDC` type is powerful, but it sits at a low level. It
owns pinned artifacts. It exposes several scoring modes. It returns raw
codes and ancestry chains. Most callers need none of that detail. They
need one function: give me a term, and tell me what it means. EideticLib
is that function.

EideticLib fixes a confidence scale. It wraps the result in a stable
`Codable` type. It enforces a strict failure policy. A caller never has
to guess whether an empty answer means "no data loaded" or "nothing
matched."

A second problem concerns privacy of memory content. Some callers
classify text a user typed into memory. That text must never leak into
LatticeLib's shared pool for novel tokens. Even one stray token would
be a problem. EideticLib exposes a `recordNovel` parameter for this
case. When a caller turns this off, the pool learns nothing from that
text. The caller still gets back the exact same anchor. One call path
can then serve both public and private text.

A third problem concerns the lattice code canon itself. FDC frames
carry a version number. Federated devices can run different frame
versions at the same time. A device might receive a lattice code from
another estate, one user's full memory store. The receiving device's
own frame may not yet recognize that code. Discarding the code would
lose information. Guessing at its meaning would break the classifier's
honesty policy. `classifyLatticeCode` sorts a code into one of three
states: malformed, known, or pending. This lets a storage layer keep a
code it cannot resolve yet. The code settles later, once a newer frame
ships.

The fourth problem is splitting text into sentences before later
pipeline stages run. Sentence boundaries look simple at first. Then
abbreviations appear, such as "Dr." or "e.g." Locale-specific
punctuation appears too. `Segmenter` solves this the way LatticeLib
solves tokenization. It offers a fast, accurate path on Apple
platforms. It offers a plain, deterministic fallback everywhere else.

## How It Works

`EideticLib.lookup(_:)` first checks `FDC.isAvailable`. If the pinned
FDC artifacts failed to load, the function stops the process with
`fatalError`. It never returns an empty or made-up answer instead. A
failed load means the binary shipped without its required data. That
is a build defect. No caller should try to recover from it at runtime.
This is a deliberate stance. MOOTx01's provenance rules treat a
placeholder identity that persists as a fabricated identity. So
EideticLib refuses to invent one.

Given available artifacts, `lookup` calls `FDC.encodeAnchor(term)`.
This call returns a lattice code and a Wikidata Q-ID. It returns `nil`
for the code when the term is UNRESOLVED. UNRESOLVED is the honest "no
answer" result. LatticeLib returns it rather than guess. EideticLib
packages a resolved result as an `Anchor` with a fixed confidence of
32, meaning medium, because FDC itself produces no calibrated
confidence score. An UNRESOLVED term becomes an `Anchor` with an empty
code, no Q-ID, and confidence 0.

`lookup(_:recordNovel:)` is the same function with one difference. It
suppresses the pool-recording side effect. Callers use this variant to
classify private memory content.

`classifyLatticeCode(_:knownCodes:)` runs independent of the FDC
engine. It checks a code string against a small grammar. The grammar
allows three digits, followed by an optional decimal extension of up
to eight digits. It uses `LatticeCodeGrammar`, a rule set that mirrors
LatticeLib's own `Code.isWellFormed(_:)`. A code that fails the grammar
is `.malformed`. A well-formed code present in the caller's
`knownCodes` set is `.known`. A well-formed code absent from that set
is `.pending`. A pending code is still accepted. It is stored and
round-tripped intact, rather than rejected.

`EideticLib.sentences(_:)` splits text into sentence substrings. On
Apple platforms, it uses `NLTokenizer`. This tool understands
abbreviations and locale-aware boundaries. On every other platform, and
whenever strict cross-platform identity matters, it uses
`sentencesByDelimiter(_:)` instead. That function splits on periods,
exclamation points, question marks, and newlines. It keeps each
terminator attached to the segment it ends. Both functions guarantee
total coverage. Every character of the input appears in exactly one
output segment. A caller can always rebuild the original text by
joining the segments back together.

## How the Pieces Fit

EideticLib is not a pipeline the way LatticeLib is. It has three
surfaces: lookup, code-state classification, and sentence
segmentation. These surfaces do not call one another. Each is an
independent, narrow entry point. Each answers one question. Each has
its own link to the rest of the SDK. `lookup` crosses into LatticeLib's
FDC engine. Code classification is a pure, dependency-free grammar
check. Sentence segmentation crosses into Apple's NaturalLanguage
framework, but only on Apple platforms.

This fan-out shape is also EideticLib's value. It is a single, stable
package. Other libraries and kits import it for a handful of small,
unrelated conveniences. They do not each need to depend on LatticeLib,
NaturalLanguage, and a code-grammar checker separately. CorpusKit, for
example, imports EideticLib only for `sentences(_:)`. CorpusKit uses
this function to split memory text into chunks before further
processing.

The library keeps the same two-language guarantee as LatticeLib. A
Swift leg serves Apple platforms. A Rust leg in `rust/` mirrors
`lookup`, `classifyLatticeCode`, and `sentencesByDelimiter` exactly.
Shared conformance vectors live in
`Tests/SharedVectors/lookup_vectors.json`. They gate every release, so
both legs agree byte for byte on every case in the fixture.

## What Ships in the Package

The package ships three Swift source files under `Sources/EideticLib/`.
It also ships an intentionally empty `Resources/` directory. EideticLib
carries no reference data of its own. LatticeLib owns the pinned
artifacts that `lookup` depends on. The package also ships the Rust
port under `rust/`.

The Rust crate is named `eidetic-lib`. It depends on the sibling
`lattice-lib` crate. This mirrors how the Swift target depends on the
LatticeLib Swift package.

Cross-language conformance rests on two checks. The first is
`Tests/SharedVectors/lookup_vectors.json`, with twenty-six vectors. The
second is the set of integration tests in `rust/tests/`.
