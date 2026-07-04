---
doc: DETAILS
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

# AriaLexiconLib Details

This document walks through every source file in the package. Read
`OVERVIEW.md` first for the big picture. Files appear here in the order a
reader builds understanding: the module surface and its one-sentence
grammar, then the noun, then the verb, then the adjective categories, and
finally the acceptance matrix that relates the noun and the verb.

## AriaLexiconLib.swift

This file provides the module surface: the public `AriaLexiconLib` enum and
the one-sentence statement of the grammar.

Swift has no module-level functions or constants, so a library that wants a
single well-known place for a top-level fact uses an empty enum as a
namespace. `AriaLexiconLib` is that namespace; it exists only to hold
`grammar`.

`AriaLexiconLib.grammar` is a constant string: "Every call is one verb
applied to a noun, optionally constrained by adjectives." This matters
because the rule is meant to be read by people, not only enforced by types.
A consumer can print this string in a log, a debugging tool, or a developer
console, and it will always match the rule the rest of the package encodes,
because it lives in the same file set and the same version as the types
that implement it.

## Noun.swift

This file provides `Noun`, the eight storage shapes the substrate persists,
and `NounRole`, how each shape relates to the one true noun of the
language, the Drawer.

A Drawer is the atomic unit of memory: one row of stored content. The other
seven cases in `Noun` — `tunnel`, `kgFact`, `vector`, `diaryEntry`,
`proposal`, `association`, and `learnedReference` — are not separate things
a caller thinks about the way it thinks about a Drawer. Each is either a
different view onto Drawer content or a record of something a verb did.
The file's opening comment is explicit that the architecture specification
calls all of these "nouns" loosely, as a storage taxonomy, but only Drawer
is a noun in ARIA's grammar sense; the rest are facets or residue.

`Noun.primary` is a static constant equal to `.drawer`. It gives code a
named way to ask "which case is the real noun" without hard-coding the
case name at every call site, so a future reader searching for "primary"
finds the one place that decision is made.

`Noun.role` is a computed property that classifies each case into one of
four `NounRole` values: `.primary` for the Drawer itself, `.rung` for a
representation of a Drawer's content (`kgFact` and `vector`), `.structure`
for an edge or event about Drawers (`tunnel`, `diaryEntry`, `association`),
and `.product` for what a verb leaves behind (`proposal`,
`learnedReference`). This grouping matters because a caller reasoning about
a storage shape — for example, deciding whether it is safe to delete —
often cares about the shape's role more than its exact case name; a rung
can usually be rebuilt from its source Drawer, while a product generally
cannot.

`NounRole` itself is a four-case, string-backed enum
(`primary`, `rung`, `structure`, `product`). Being string-backed means it
serializes to a stable, human-readable form rather than a raw integer, which
matters for any log or wire format that includes it.

## Verb.swift

This file provides `Verb`, the nine actions the substrate supports, and
`Flow`, who initiates each one.

The nine verbs are `capture`, `reanchor`, `mutate`, `withdraw`, `expunge`,
`recall`, `propose`, `associate`, and `learn`. The file's opening comment
states the design rule directly: this count is fixed by specification
invariant I-7, and a new kind of domain operation is expected to compose
these nine rather than add a tenth. Keeping the verb set closed is what
lets every kit share one small vocabulary instead of each kit growing its
own private verbs over time.

`Verb.flow` is a computed property returning a `Flow` value that answers
"who calls this verb." Six verbs — `capture`, `reanchor`, `mutate`,
`withdraw`, `expunge`, and `recall` — are `.callerDriven`: an application
calls them directly, synchronously, when it wants something to happen. Two
verbs, `propose` and `associate`, are `.substrateDriven`: they are emitted
by the system's own background processes on their own schedule, not called
directly by application code. One verb, `learn`, is `.groundingDriven`: it
is the verb that brings authoritative outside reference material into the
system, distinct from both a direct caller action and an internal
background signal. Separating these three flows matters for anyone writing
code against the substrate, because a caller-driven verb needs a call site,
while a substrate-driven or grounding-driven verb needs a listener instead.

`Flow` is a three-case, string-backed enum (`callerDriven`,
`substrateDriven`, `groundingDriven`).

## Adjective.swift

This file provides `Adjective`, the four categories that describe any row
of memory regardless of which noun or verb produced it.

The four categories are `state` (where the row sits in its lifecycle, such
as active, pending, or superseded), `trust` (how the content was
established, such as verbatim, observed, or derived), `sensitivity` (how
exposed the content may be), and `exportability` (whether the content may
leave the access perimeter — private or public). The count of four is
fixed by specification invariant I-8, the same way the verb count is fixed
by I-7.

The file's comment explains an important design boundary: `Adjective` only
names the four categories, not the specific values inside each one. Every
row carries some value in each category no matter its storage shape, which
is why the categories are described as "cross-noun." But how those values
are packed and represented — the actual list of lifecycle states, for
example — is treated as a bitmap-layout detail, and that detail is reified
in a separate kit, LocusKit, instead of here. Keeping the category names in
one place and the value lists in another means the two cannot silently
drift apart by being maintained twice; only one of them needs to change
when a new value is added to a category.

## Acceptance.swift

This file provides `Acceptance`, the verb-noun acceptance matrix: the one
place the package states which of the nine verbs make sense for each of
the eight storage shapes.

The matrix is hand-authored data, not derived from `Noun.role` or
`Verb.flow`. For example, `Proposal`'s role is "product," yet it does
accept the verb that creates it, `propose`, along with the ordinary
lifecycle verbs (`mutate`, `withdraw`, `expunge`, `recall`). The file's
opening comment frames this precisely: in ARIA's grammar the matrix reads
as "which actions apply to this shape," not as a competition between
nouns, and it is expressed as data specifically so that a conformance test
can check the Swift and Rust implementations agree cell by cell.

`Acceptance.verbs(for:)` takes a `Noun` and returns the `Set<Verb>` it
accepts. A Drawer accepts six verbs (every caller-driven verb except none
— it is the fullest row in the matrix). A Tunnel accepts five, omitting
`reanchor`. A knowledge-graph fact accepts four, omitting both `capture`
and `reanchor`, because a fact is not captured directly the way a Drawer
or Tunnel is. A Vector accepts none: the comment explains that vectors are
substrate-managed, meaning an internal process derives and maintains them,
so no caller ever addresses one with a verb directly. A diary entry accepts
only `recall`, since it is a read-only record. A Proposal accepts
`propose` plus the lifecycle verbs, and an Association accepts `associate`
plus most of the lifecycle verbs (it omits `withdraw`). A learned reference
accepts `learn` plus the lifecycle verbs. This function matters because it
is the only place any of these facts is stated; a caller that needs to know
"can I do X to a Y" asks this function instead of re-deriving the answer
from role or flow, which would sometimes give the wrong answer, as the
Proposal example shows.

`Acceptance.accepts(_:_:)` takes a noun and a verb and returns whether the
verb is a member of that noun's accepted set. It is a thin convenience over
`verbs(for:)`, provided because most call sites want a yes-or-no answer for
one specific pair rather than the whole set, and spelling that check inline
at every call site would risk a typo that a shared function cannot make.

## Rust Port and Conformance

The `rust/` directory contains the second leg of the library, `lib.rs`, a
single-file crate named `aria-lexicon-lib`. It mirrors every Swift type:
`Noun`, `NounRole`, `Verb`, `Flow`, and `Adjective` as Rust enums, plus
`accepted_verbs` and `accepts` as the Rust equivalents of
`Acceptance.verbs(for:)` and `Acceptance.accepts(_:_:)`. Swift's string-backed
enum cases are already lowerCamelCase and serve as their own wire strings.
Rust instead declares PascalCase variants by convention and adds an
`as_str()` method, plus `serde` attributes, that reproduce the identical
lowerCamelCase wire strings for JSON — for example, `Noun::KgFact.as_str()`
returns `"kgFact"`, matching Swift's `Noun.kgFact.rawValue` exactly.

The crate's own test module is the conformance suite: it checks the verb
count is nine and the adjective count is four, checks every wire string
against its expected value, checks that `serde` serialization matches
`as_str()` for every case, and re-states the entire acceptance matrix cell
by cell, including the harder-to-guess cases such as a Proposal not
accepting `associate` and an Association not accepting `propose`. Because
the two languages cannot share source code for this vocabulary, this test
module is the only mechanism that catches the two ports drifting apart:
when you change `Acceptance.swift`, `Noun.swift`, `Verb.swift`, or
`Adjective.swift`, you must make the matching edit in `rust/src/lib.rs` and
rerun both test suites, or a silent disagreement ships.
