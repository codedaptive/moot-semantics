---
doc: AGENT_MAP
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

# AGENT_MAP | AriaLexiconLib

PURPOSE: the reified ARIA grammar, one noun (Drawer), nine verbs, four adjective categories, and the verb-noun acceptance matrix, as pure data. No behavior, no I/O, no dependencies. Canonical prose statement lives outside this package in ARIA_LEXICON.md; this module is that statement made first-class in code so it can be conformance-checked.

DEPS: imports nothing (zero-dependency, foundational leaf package; Swift stdlib only). Imported by: none currently declare a package/`import` dependency within this SDK checkout. LocusKit (moot-memory/packages/kits/LocusKit) references `AriaLexiconLib`'s types by name in doc comments (Acceptance.swift, Verb.swift, Noun.swift) as the conceptual source of truth for its own Frames/Association/Proposal/LearnedReference/EstateVerbs types, but does NOT declare it as a Package.swift dependency or `import` it as of this commit: treat as documentation-level, not compiled, coupling. Package header names VectorKit and CorpusKit as intended future conformers. Rust port: `rust/` (crate `aria-lexicon-lib`, single file `src/lib.rs`) mirrors every type and the acceptance matrix by hand; its own `#[cfg(test)]` module is the conformance gate (wire-string + matrix parity vs. this Swift package).

ENTRY POINTS (most callers need only these):
- Acceptance.swift:36 `Acceptance.accepts(_ noun: Noun, _ verb: Verb) -> Bool` | the one call most callers want: "is this operation allowed on this shape"
- Acceptance.swift:14 `Acceptance.verbs(for noun: Noun) -> Set<Verb>` | full accepted-verb set for a shape
- AriaLexiconLib.swift:15 `AriaLexiconLib.grammar` | the one-sentence rule as a displayable string

## Symbol Table

### Module | AriaLexiconLib.swift
- :13 `enum AriaLexiconLib` | empty namespace enum
- :15 `static let grammar: String` | "Every call is one verb applied to a noun, optionally constrained by adjectives."

### Noun | Noun.swift
- :14 `enum Noun: String, CaseIterable, Sendable, Codable` | 8 storage shapes; rawValue IS the wire string (lowerCamelCase case names, no explicit `=`)
- :15–22 cases: `drawer`, `tunnel`, `kgFact`, `vector`, `diaryEntry`, `proposal`, `association`, `learnedReference` (declaration order = Rust `Noun::ALL` order)
- :27 `static let primary: Noun = .drawer` | the one true noun of the language
- :31 `var role: NounRole` | computed; drawer→primary, kgFact/vector→rung, tunnel/diaryEntry/association→structure, proposal/learnedReference→product
- :45 `enum NounRole: String, CaseIterable, Sendable, Codable` | :47 `primary`, :49 `rung`, :51 `structure`, :53 `product`

### Verb | Verb.swift
- :9 `enum Verb: String, CaseIterable, Sendable, Codable` | 9 actions, count fixed by spec invariant I-7
- :10–18 cases: `capture`, `reanchor`, `mutate`, `withdraw`, `expunge`, `recall`, `propose`, `associate`, `learn` (declaration order = Rust `Verb::ALL` order)
- :21 `var flow: Flow` | computed; capture/reanchor/mutate/withdraw/expunge/recall→callerDriven, propose/associate→substrateDriven, learn→groundingDriven
- :37 `enum Flow: String, CaseIterable, Sendable, Codable` | :38 `callerDriven`, :39 `substrateDriven`, :40 `groundingDriven`

### Adjective | Adjective.swift
- :13 `enum Adjective: String, CaseIterable, Sendable, Codable` | 4 cross-noun categories, count fixed by spec invariant I-8
- :17 `state` | lifecycle position (active/pending/contested/superseded/decayed/withdrawn/expired/rejected/accepted/tombstoned: VALUES live in LocusKit, not here)
- :20 `trust` | provenance (verbatim/observed/imported/proposed/derived/canonical: values in LocusKit)
- :23 `sensitivity` | exposure level (normal/elevated/restricted/secret: values in LocusKit)
- :26 `exportability` | perimeter crossing (private/public: values in LocusKit)

### Acceptance matrix | Acceptance.swift
- :10 `enum Acceptance` | namespace for the matrix
- :14 `static func verbs(for noun: Noun) -> Set<Verb>` | hand-authored per-noun set; SEE ENTRY POINTS
- :16 drawer → {capture, reanchor, mutate, withdraw, expunge, recall} (6, fullest row)
- :18 tunnel → {capture, mutate, withdraw, expunge, recall} (5, no reanchor)
- :20 kgFact → {mutate, withdraw, expunge, recall} (4, no capture/reanchor)
- :22 vector → {} (0: substrate-managed, never verb-addressable directly)
- :24 diaryEntry → {recall} (1, read-only)
- :26 proposal → {propose, mutate, withdraw, expunge, recall} (5; role is "product" but DOES accept propose)
- :28 association → {associate, mutate, expunge, recall} (4; no withdraw)
- :30 learnedReference → {learn, mutate, withdraw, expunge, recall} (5)
- :36 `static func accepts(_ noun: Noun, _ verb: Verb) -> Bool` | `verbs(for:).contains(verb)`; SEE ENTRY POINTS

### Rust port | rust/src/lib.rs (mirror, not a Swift symbol)
- `Noun`/`NounRole`/`Verb`/`Flow`/`Adjective` | PascalCase enum variants + `as_str()` producing the identical lowerCamelCase wire strings as the Swift rawValues
- `accepted_verbs(Noun) -> Vec<Verb>` / `accepts(Noun, Verb) -> bool` | same matrix, hand-mirrored
- `#[cfg(test)] mod tests` | conformance gate: verb/adjective counts, full matrix cell-by-cell, wire-string parity, serde round-trip

## INVARIANTS / GOTCHAS

- NO BEHAVIOR. This package is pure data (enums, one constant string, one lookup table). Do not add I/O, logging, or business logic here; new domain operations compose the nine existing verbs instead of extending the vocabulary.
- Verb count is pinned at 9 (spec invariant I-7). Adjective category count is pinned at 4 (spec invariant I-8). Both are spec-level invariants, not incidental: changing either is a specification amendment, not a routine edit.
- `Acceptance`'s matrix is HAND-AUTHORED, not derived from `Noun.role` or `Verb.flow`. Do not "simplify" it by deriving from role/flow: the Proposal row (role=`product`, yet accepts `.propose`) and the Association row (role=`structure`, accepts `.associate` but not `.propose`) are real exceptions that a derived rule would get wrong.
- `Adjective` names categories only. The value lists inside each category (the 10 lifecycle states, 6 trust levels, 4 sensitivity levels, 2 exportability levels) live in LocusKit (moot-memory), not here: by design, so the two never fork by being defined twice. Do not add value enums to this file.
- Zero dependencies is load-bearing: this package must stay leaf-level (imports nothing) so any kit, including ones layered far apart, like LocusKit, VectorKit, and CorpusKit, can depend on it without a cycle.
- Swift enum rawValues ARE the wire strings (case names are already lowerCamelCase; no explicit `= "..."` needed). Rust mirrors with PascalCase variants + `as_str()`/`serde(rename_all = "camelCase")` to reproduce the identical strings. Any edit to a Swift case name, or to Rust's `as_str()`/serde attributes, that the other side does not mirror breaks wire conformance silently unless the Rust test suite is rerun.
- Two independent implementations, no shared codegen: editing `Acceptance.swift`, `Noun.swift`, `Verb.swift`, or `Adjective.swift` requires a matching hand-edit to `rust/src/lib.rs` (types, `as_str()`, and `accepted_verbs`/`accepts`) plus a rerun of the Rust `#[cfg(test)]` suite. There is no build step that generates one from the other.
- Canonical source-of-truth order: the prose document ARIA_LEXICON.md (external to this package, not present in this repo checkout) states the grammar; this Swift package is the code reification of that document; the Rust crate is a conformance-gated mirror of this package. When in doubt about intent, ARIA_LEXICON.md wins.
- No compiled consumer exists yet within this SDK checkout: verify current importers before assuming call sites exist. LocusKit's references are doc-comment citations, not `import` statements or Package.swift dependencies, as of this commit.
