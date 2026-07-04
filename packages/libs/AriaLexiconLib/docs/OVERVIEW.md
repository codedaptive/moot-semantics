---
doc: OVERVIEW
package: AriaLexiconLib
repo: moot-semantics
authored_commit: 021ea704162f86fccea2f030ea9419dacc30a345
authored_date: 2026-07-04
sources:
  - path: Sources/AriaLexiconLib/Acceptance.swift
    blob: 095b16e63587937c22911a7bada283afdf5aea1a
  - path: Sources/AriaLexiconLib/Adjective.swift
    blob: 4c74c273c5fbf1c863f478de3287133119780936
  - path: Sources/AriaLexiconLib/AriaLexiconLib.swift
    blob: 6a7f1d2b66f7fb9b2635befacdd44f3c3212217b
  - path: Sources/AriaLexiconLib/Noun.swift
    blob: 539bf0460df8d6f60d5e63c11a7706c4c8557eb3
  - path: Sources/AriaLexiconLib/Verb.swift
    blob: 7ecf10b17758b23b7652c47879c2b353d28f5965
---

# AriaLexiconLib Overview

## What This Library Does

AriaLexiconLib names the words that every MOOTx01 kit uses to describe an
action on memory. Inside MOOTx01, that fixed vocabulary is called ARIA. ARIA
states a simple grammar: every call to the substrate is one verb applied to
a noun, optionally constrained by adjectives. AriaLexiconLib turns that
sentence into code, so a program can check whether a given verb is even
meaningful for a given noun before it tries the call.

The library has one noun, the Drawer, which is the atomic unit of memory.
Every other storage shape the substrate persists — a knowledge-graph fact, a
vector, a diary entry — is either a facet of a Drawer or something a verb
leaves behind. The library has nine verbs, such as capture, recall, and
withdraw, and four adjective categories, such as trust and sensitivity, that
describe a row without changing which noun or verb it is. A small acceptance
matrix records which verbs apply to which storage shapes.

## The Problem It Solves

MOOTx01 is an on-device AI memory system. It stores what an AI observes over
time and helps the AI recall it later. Many separate packages — the SDK
described at the top of this repository set calls them kits — build on top
of this vocabulary: one kit stores knowledge-graph facts, another stores
vector embeddings, another manages the estate's day-to-day operations. If
each kit invented its own names for "add a memory" or "remove a memory," a
caller would need to learn a different vocabulary per kit, and it would
become easy to call an operation that does not make sense for a given
storage shape, such as asking a vector to be captured directly instead of
letting the substrate manage it.

AriaLexiconLib fixes the vocabulary once, in one place, as data rather than
as prose scattered across each kit's documentation. Every kit conforms to
it: a kit's API surface uses these same noun and verb names, and its
allowed-operations logic reads its answers from the acceptance matrix
instead of restating the rule. The vocabulary also crosses languages: the
library ships a Rust port so that non-Swift consumers speak the identical
words, checked one against the other by shared tests.

The canonical statement of the grammar is a prose document, ARIA_LEXICON.md,
maintained outside this package. AriaLexiconLib is that document made
first-class in code, so an editor, a compiler, and a conformance test can
all point at the same set of names.

## How It Works

The library defines four small pieces of vocabulary and one rule that
relates two of them.

`Noun` lists the eight storage shapes the substrate persists, and marks the
Drawer as the one true noun of the language. The other seven are either a
"rung" (a representation of a Drawer's content, such as a vector), a piece
of "structure" (an edge or event about Drawers, such as an association), or
a "product" (what a verb leaves behind, such as a proposal).

`Verb` lists the nine actions the substrate supports: capture, reanchor,
mutate, withdraw, expunge, recall, propose, associate, and learn. Each verb
also names who initiates it. Six verbs are caller-driven, meaning an
application calls them directly. Two are substrate-driven, meaning an
internal background process emits them on its own schedule. One,
`learn`, is grounding-driven: it is how authoritative outside reference
material enters the system.

`Adjective` lists the four categories that describe a row no matter which
noun or verb touched it: state (where the row sits in its lifecycle, such as
active or superseded), trust (how the content was established, such as
verbatim or derived), sensitivity (how exposed the content may be), and
exportability (whether the content may leave the access perimeter). This
library only names the four categories. The specific values inside each
category — for example, the full list of lifecycle states — are a detail of
how the data is packed into storage, and that detail lives in a separate
kit, LocusKit, so the two never drift out of agreement with each other by
being defined twice.

`Acceptance` is the one rule that connects `Noun` and `Verb`. For each of
the eight nouns, it lists exactly which of the nine verbs make sense. A
Drawer accepts six verbs, including capture and recall. A Vector accepts
none, because the substrate manages vectors on its own; nothing calls a verb
on one directly. This matrix is hand-authored, not derived automatically
from any other rule in the library, so it is the single place a caller or a
test checks "is this operation allowed here."

## How the Pieces Fit

Figure 1 shows how the four vocabulary types and the acceptance matrix
relate to each other and to the Rust port.

![Figure 1. Topology of AriaLexiconLib](topology.svg)

*Figure 1. Topology of AriaLexiconLib. `Noun` and `Verb` feed the
`Acceptance` matrix; `Adjective` stands beside them as an independent,
cross-noun category list. The dashed regions mark the Rust mirror and the
downstream packages that conform to this vocabulary without this package
depending on them.*

`AriaLexiconLib.grammar` states the one-sentence rule in plain English, as a
constant string, so a consumer can display or log the rule itself rather
than reconstruct it from the types. `Noun` and `Verb` are independent
enumerations; neither refers to the other. `Acceptance.verbs(for:)` and
`Acceptance.accepts(_:_:)` are the only functions in the package, and both
simply look up a fixed table keyed by a noun. `Adjective` does not
participate in the acceptance rule at all — an adjective always applies to
a row regardless of its noun or the verb that produced it.

The Rust port in `rust/` mirrors every type and the acceptance matrix by
hand, because the two languages cannot share source code. A build-time
conformance suite inside the Rust crate re-asserts every wire string and
every cell of the acceptance matrix against the values recorded here, so a
change to either side that the other side misses fails a test rather than
shipping a silent disagreement.

## What Ships in the Package

The package ships five Swift source files, no bundled data files, and the
Rust port in `rust/`. There are no pinned artifacts and no build-time
tooling: every value in the package is a literal written directly in the
source, because the vocabulary is small and fixed by specification rather
than computed or trained. The verb count is fixed at nine and the adjective
category count at four; both counts are named spec invariants, so growing
either number is a specification change, not a routine code edit.
