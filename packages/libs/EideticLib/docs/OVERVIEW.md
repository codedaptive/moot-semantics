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

EideticLib is a thin, deterministic front door onto LatticeLib's
classification engine. A caller hands it a term and gets back an
`Anchor`: a lattice code, the dominant Wikidata Q-ID of the term, a
confidence value, and the data version that produced the answer. A
Wikidata Q-ID is a public, language-neutral identifier for one
concept, such as `Q144` for "dog." EideticLib holds no reference data
of its own. Every lookup delegates to `FDC.encodeAnchor`, the runtime
entry point of LatticeLib's Frame-Directed Classification (FDC)
engine.

Alongside lookup, EideticLib carries two small, self-contained
utilities that other parts of the SDK need: a check for whether a
lattice code string is well-formed and known, and a function that
splits text into sentences. Neither utility depends on the FDC engine.
All three surfaces share one purpose: give callers a deterministic
answer without asking them to understand LatticeLib's internals.

## The Problem It Solves

LatticeLib's `FDC` type is a powerful but low-level engine: it owns
pinned artifacts, exposes several scoring modes, and returns raw
codes and ancestry chains. Most callers do not need any of that. They
need one function: give me a term, tell me what it is about. EideticLib
is that function. It fixes a confidence scale, wraps the result in a
stable `Codable` type, and enforces a strict failure policy so callers
never have to guess whether an empty answer means "no data loaded" or
"nothing matched."

A second, related problem is memory content privacy. Some callers
classify user-supplied memory content — private text that must never
leak into LatticeLib's shared novel-token learning pool, even by
accident. EideticLib exposes a `recordNovel` parameter that turns that
accumulation off while returning the exact same anchor, so a capture
path can classify text safely without a second, divergent code path.

A third problem concerns the lattice code canon itself. FDC frames are
versioned, and federated devices can be running against different
frame versions at different times. A device might receive a lattice
code from another estate — one user's complete memory store — that its
own local frame does not yet recognize. Discarding that code would
lose information; guessing at its meaning would violate the classifier's
honesty. `classifyLatticeCode` sorts a code string into one of three
states — malformed, known, or pending — so storage layers can accept and
round-trip a code they cannot yet resolve, and settle it later when a
newer frame ships.

The fourth problem is splitting text into sentences before other
pipeline stages run. Sentence boundaries look simple until abbreviations
("Dr.", "e.g.") and locale-specific punctuation appear. `Segmenter`
answers this the same way LatticeLib answers tokenization: a fast,
higher-accuracy path on Apple platforms, and a plain, deterministic
fallback everywhere else.

## How It Works

`EideticLib.lookup(_:)` first checks `FDC.isAvailable`. If the pinned
FDC artifacts failed to load, the function terminates the process with
`fatalError` rather than returning an empty or fabricated answer. A
failed load means the binary shipped without its required data, which
is a build defect, not a condition any caller should recover from at
runtime. This is a deliberate stance: MOOTx01's provenance rules treat
a persistent placeholder identity as a fabricated identity, so
EideticLib refuses to manufacture one.

Given available artifacts, `lookup` calls `FDC.encodeAnchor(term)`,
which returns a lattice code and a Wikidata Q-ID, or `nil` for the
code when the term is UNRESOLVED — the honest "no answer" result that
LatticeLib returns rather than guess. EideticLib packages a resolved
result as an `Anchor` with a fixed confidence of 32 (medium), because
FDC produces no calibrated confidence score of its own. An UNRESOLVED
term becomes an `Anchor` with an empty code, no Q-ID, and confidence 0.
`lookup(_:recordNovel:)` is the same function with the pool-recording
side effect suppressed, for callers classifying private memory content.

`classifyLatticeCode(_:knownCodes:)` runs independently of the FDC
engine. It checks the code string against a small grammar — three
digits, an optional decimal extension of up to eight digits — using
`LatticeCodeGrammar`, a self-contained rule set that mirrors LatticeLib's
own `Code.isWellFormed(_:)`. A code that fails the grammar is
`.malformed`. A well-formed code present in the caller-supplied
`knownCodes` set is `.known`. A well-formed code absent from that set
is `.pending`: accepted, storable, and round-tripped intact rather than
rejected.

`EideticLib.sentences(_:)` splits text into sentence substrings. On
Apple platforms it uses `NLTokenizer`, which understands abbreviations
and locale-aware boundaries. On every other platform, and whenever
strict cross-platform identity is required, `sentencesByDelimiter(_:)`
splits on periods, exclamation points, question marks, and newlines,
keeping the terminator attached to the segment it ends. Both functions
guarantee total coverage: every character of the input appears in
exactly one output segment, so a caller can always reconstruct the
original text by joining the segments back together.

## How the Pieces Fit

EideticLib is not a pipeline the way LatticeLib is. Its three surfaces
— lookup, code-state classification, and sentence segmentation — do not
call one another. Each is an independent, narrow entry point that
answers one question, and each has its own relationship to the rest of
the SDK: `lookup` crosses into LatticeLib's FDC engine, code
classification is a pure, dependency-free grammar check, and sentence
segmentation crosses into Apple's NaturalLanguage framework only on
Apple platforms.

That fan-out shape is also EideticLib's value: it is a single, stable
package that other libraries and kits import for a handful of small,
unrelated conveniences, so they do not each have to depend on
LatticeLib, NaturalLanguage, and a code-grammar checker separately.
CorpusKit, for example, imports EideticLib only for `sentences(_:)`,
to segment memory text into chunks before further processing.

The library keeps the same two-language guarantee as LatticeLib. A
Swift leg serves Apple platforms, and a Rust leg in `rust/` mirrors
`lookup`, `classifyLatticeCode`, and `sentencesByDelimiter` exactly.
Shared conformance vectors in `Tests/SharedVectors/lookup_vectors.json`
gate every release so both legs agree byte for byte on every case in
the fixture.

## What Ships in the Package

The package ships three Swift source files under `Sources/EideticLib/`,
an intentionally empty `Resources/` directory (EideticLib carries no
reference data of its own; LatticeLib owns the pinned artifacts that
`lookup` depends on), and the Rust port under `rust/`. The Rust crate,
`eidetic-lib`, depends on the sibling `lattice-lib` crate the same way
the Swift target depends on the LatticeLib Swift package. Cross-language
conformance is enforced by `Tests/SharedVectors/lookup_vectors.json`
(26 vectors) and the integration tests in `rust/tests/`.
