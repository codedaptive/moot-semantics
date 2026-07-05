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
reader builds understanding. First comes the module surface and its
one-sentence grammar. Then comes the noun. Then comes the verb. Then comes
the adjective categories. Last comes the acceptance matrix that relates
the noun and the verb.

## AriaLexiconLib.swift

This file provides the module surface. It holds the public
`AriaLexiconLib` enum and the one-sentence statement of the grammar.

Swift has no module-level functions or constants. A library that wants a
single well-known place for a top-level fact uses an empty enum as a
namespace instead. `AriaLexiconLib` is that namespace. It exists only to
hold `grammar`.

`AriaLexiconLib.grammar` is a constant string: "Every call is one verb
applied to a noun, optionally constrained by adjectives." This matters
because people need to read the rule, not only have types enforce it. A
consumer can print this string in a log, a debugging tool, or a developer
console. It will always match the rule the rest of the package encodes.
It lives in the same file set and the same version as the types that
implement it.

## Noun.swift

This file provides `Noun`, the eight storage shapes the substrate
persists. It also provides `NounRole`, how each shape relates to the one
true noun of the language, the Drawer.

A Drawer is the atomic unit of memory: one row of stored content. The
other seven cases in `Noun` are `tunnel`, `kgFact`, `vector`,
`diaryEntry`, `proposal`, `association`, and `learnedReference`. Each of
these seven is not a separate thing a caller thinks about the way it
thinks about a Drawer. Each is either a different view onto Drawer
content, or a record of something a verb did. The file's opening comment
is explicit. The architecture specification calls all of these "nouns"
loosely, as a storage taxonomy. Only Drawer is a noun in ARIA's grammar
sense. The rest are facets or residue.

`Noun.primary` is a static constant equal to `.drawer`. It gives code a
named way to ask which case is the real noun. It avoids hard-coding the
case name at every call site. A future reader searching for "primary"
finds the one place that decision is made.

`Noun.role` is a computed property. It classifies each case into one of
four `NounRole` values. `.primary` covers the Drawer itself. `.rung`
covers a representation of a Drawer's content: `kgFact` and `vector`.
`.structure` covers an edge or event about Drawers: `tunnel`,
`diaryEntry`, and `association`. `.product` covers what a verb leaves
behind: `proposal` and `learnedReference`. This grouping matters because a
caller often cares about a shape's role more than its exact case name.
For example, a caller deciding whether it is safe to delete something
cares about role. A rung can usually be rebuilt from its source Drawer. A
product generally cannot.

`NounRole` itself is a four-case, string-backed enum: `primary`, `rung`,
`structure`, `product`. Being string-backed means it serializes to a
stable, human-readable form, rather than a raw integer. This matters for
any log or wire format that includes it.

## Verb.swift

This file provides `Verb`, the nine actions the substrate supports. It
also provides `Flow`, who initiates each one.

The nine verbs are `capture`, `reanchor`, `mutate`, `withdraw`,
`expunge`, `recall`, `propose`, `associate`, and `learn`. The file's
opening comment states the design rule directly. This count is fixed by
specification invariant I-7. A new kind of domain operation is expected
to compose these nine, rather than add a tenth. Keeping the verb set
closed lets every kit share one small vocabulary. Otherwise each kit
would grow its own private verbs over time.

`Verb.flow` is a computed property. It returns a `Flow` value that
answers who calls this verb. Six verbs are `.callerDriven`: `capture`,
`reanchor`, `mutate`, `withdraw`, `expunge`, and `recall`. An application
calls each of these directly and synchronously, when it wants something
to happen. Two verbs are `.substrateDriven`: `propose` and `associate`.
The system's own background processes emit these on their own schedule.
No application code calls them directly. One verb, `learn`, is
`.groundingDriven`. It is the verb that brings authoritative outside
reference material into the system. It differs from both a direct caller
action and an internal background signal. Separating these three flows
matters for anyone writing code against the substrate. A caller-driven
verb needs a call site. A substrate-driven or grounding-driven verb needs
a listener instead.

`Flow` is a three-case, string-backed enum: `callerDriven`,
`substrateDriven`, `groundingDriven`.

## Adjective.swift

This file provides `Adjective`, the four categories that describe any row
of memory, regardless of which noun or verb produced it.

The four categories are as follows. `state` describes where the row sits
in its lifecycle, such as active, pending, or superseded. `trust`
describes how the content was established, such as verbatim, observed, or
derived. `sensitivity` describes how exposed the content may be.
`exportability` describes whether the content may leave the access
perimeter, such as private or public. The count of four is fixed by
specification invariant I-8, the same way the verb count is fixed by I-7.

The file's comment explains an important design boundary. `Adjective`
only names the four categories. It does not name the specific values
inside each one. Every row carries some value in each category, no
matter its storage shape. This is why the categories are described as
"cross-noun." How those values are packed and represented is a different
matter. The actual list of lifecycle states, for example, is treated as
a bitmap-layout detail. That detail is reified in a separate kit,
LocusKit, instead of here. Keeping the category names in one place and
the value lists in another means the two cannot silently drift apart by
being maintained twice. Only one of them needs to change when a new
value is added to a category.

## Acceptance.swift

This file provides `Acceptance`, the verb-noun acceptance matrix. It is
the one place the package states which of the nine verbs make sense for
each of the eight storage shapes.

The matrix is hand-authored data. No other rule in the package, neither
`Noun.role` nor `Verb.flow`, derives it. For example, `Proposal`'s role
is `product`. Yet it accepts the verb that creates it, `propose`. It also
accepts the ordinary lifecycle verbs: `mutate`, `withdraw`, `expunge`,
and `recall`. The file's opening comment frames this precisely. In ARIA's
grammar the matrix reads as which actions apply to this shape. It is not
a competition between nouns. It is expressed as data specifically so a
conformance test can check the Swift and Rust implementations agree cell
by cell.

`Acceptance.verbs(for:)` takes a `Noun` and returns the `Set<Verb>` it
accepts. A Drawer accepts six verbs. It is the fullest row in the
matrix, since every caller-driven verb applies to it. A Tunnel accepts
five verbs. It omits `reanchor`. A knowledge-graph fact accepts four
verbs. It omits both `capture` and `reanchor`. A fact is not captured
directly the way a Drawer or a Tunnel is. A Vector accepts no verbs. The
comment explains that vectors are substrate-managed. An internal process
derives and maintains them. No caller ever addresses one with a verb
directly. A diary entry accepts only `recall`, since it is a read-only
record. A Proposal accepts `propose` plus the lifecycle verbs. An
Association accepts `associate` plus most of the lifecycle verbs. It
omits `withdraw`. A learned reference accepts `learn` plus the lifecycle
verbs. This function matters because it is the only place any of these
facts is stated. A caller that needs to know whether it can do something
to a shape asks this function. It does not re-derive the answer from role
or flow. Re-deriving would sometimes give the wrong answer, as the
Proposal example shows.

`Acceptance.accepts(_:_:)` takes a noun and a verb. It returns whether the
verb is a member of that noun's accepted set. It is a thin convenience
over `verbs(for:)`. Most call sites want a yes-or-no answer for one
specific pair, rather than the whole set. Spelling that check inline at
every call site would risk a typo that a shared function cannot make.

## Rust Port and Conformance

The `rust/` directory contains the second leg of the library: `lib.rs`, a
single-file crate named `aria-lexicon-lib`. It mirrors every Swift type
as Rust enums: `Noun`, `NounRole`, `Verb`, `Flow`, and `Adjective`. It
also adds `accepted_verbs` and `accepts` as the Rust equivalents of
`Acceptance.verbs(for:)` and `Acceptance.accepts(_:_:)`. Swift's
string-backed enum cases are already lowerCamelCase. They serve as their
own wire strings. Rust instead declares PascalCase variants by
convention. It adds an `as_str()` method, plus `serde` attributes, that
reproduce the identical lowerCamelCase wire strings for JSON. For
example, `Noun::KgFact.as_str()` returns `"kgFact"`. This exactly matches
Swift's `Noun.kgFact.rawValue`.

The crate's own test module is the conformance suite. It checks the verb
count is nine and the adjective count is four. It checks every wire
string against its expected value. It checks that `serde` serialization
matches `as_str()` for every case. It also re-states the entire
acceptance matrix, cell by cell, including the harder-to-guess cases. One
such case is a Proposal not accepting `associate`. Another is an
Association not accepting `propose`. The two languages cannot share
source code for this vocabulary, so this test module is the only
mechanism that catches the two ports drifting apart.

Four files must stay in sync: `Acceptance.swift`, `Noun.swift`,
`Verb.swift`, and `Adjective.swift`. Any edit to one of them requires a
matching edit in `rust/src/lib.rs`. It also requires a rerun of both test
suites. Otherwise a silent disagreement ships.
