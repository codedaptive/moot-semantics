---
doc: AGENT_MAP
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

# AGENT_MAP: EideticLib

PURPOSE: thin deterministic facade over LatticeLib's FDC engine (term → Anchor{code, wikidataQID, confidence, dataVersion}) plus content-aware text/code anchoring and two unrelated small utilities: lattice-code shape/known-state classification, and sentence segmentation. Carries no reference data of its own.

DEPS: imports LatticeLib (FDC.encodeAnchor/isAvailable/dataVersion), NaturalLanguage (Apple-only NLTokenizer sentence path), Foundation. Imported by: CorpusKit (moot-memory kit, `EideticLib.sentences` only) and, per Package.swift header comment, NeuronKit (not present in these four venue repos: likely mootx01-ee/ce, out of scope here). Rust port in rust/ (crate `eidetic-lib`) mirrors lookup, classifyLatticeCode, and the delimiter segmenter exactly; depends on sibling `lattice-lib` crate. No Rust counterpart to the NLTokenizer path (Apple-only, excluded from cross-leg parity by the apple-nlp-accel constitutional constraint C-2).

ENTRY POINTS (most callers need only these):
- EideticLib.swift:67 `EideticLib.lookup(_ term: String) -> Anchor`: main lookup; fatalError if FDC.isAvailable is false
- EideticLib.swift:118 `EideticLib.lookup(_:recordNovel:)`: same, recordNovel:false = privacy seam (no LatticeLib pool writes)
- EideticLib.swift:40 `EideticLib.classifyLatticeCode(_:knownCodes:) -> LatticeCodeState`: malformed/known/pending
- Segmenter.swift:47 `EideticLib.sentences(_ text: String) -> [Substring]`: platform-routed sentence split

## Symbol Table

- EideticLib.swift `enum EideticContentKind`: `.text` or `.code`.
- EideticLib.swift `lookup(_:contentKind:recordNovel:)`: code anchors at FDC `005`; a decisive local language match supplies its pinned Q-ID.

### Lookup surface: EideticLib.swift
- :20 `enum EideticLib`: namespace; holds no reference data, delegates entirely to LatticeLib.FDC
- :23 `version = "0.1.0"`: module version string
- :40 `classifyLatticeCode(_:knownCodes:) -> LatticeCodeState`: grammar check via LatticeCodeGrammar, then knownCodes membership
- :67 `lookup(_:) -> Anchor`: guards FDC.isAvailable (else fatalError); calls FDC.encodeAnchor(term); resolved→confidence 32, UNRESOLVED→empty code/confidence 0
- :118 `lookup(_:recordNovel:) -> Anchor`: identical Anchor to lookup(_:); recordNovel:false suppresses LatticeLib novel-token pool accumulation; used by GLK capture intake (EncodeIntake) so private memory content never reaches the pool
- :147 `struct Anchor: Equatable, Sendable, Codable`: code (empty=UNRESOLVED) / wikidataQID (String?) / confidence (UInt8: 0/16/32/48/56=null/low/medium/high/verified; EideticLib only emits 0 or 32) / dataVersion (FDC signatures version); byte-identical JSON shape to Rust `Anchor`

### Lattice-code state: LatticeCodeState.swift
- :27 `enum LatticeCodeState: Sendable, Hashable, Codable`: .malformed(String) / .known(String) / .pending(String)
- :46 `rawCode: String`: original input regardless of case (storage round-trip without unpacking)
- :56 `isWellFormed: Bool`: false only for .malformed
- :72 `enum LatticeCodeGrammar`: dependency-free; parallel reimplementation of LatticeLib's `Code.isWellFormed(_:)`, kept in agreement by shared conformance tests, NOT shared code
- :77 `maxExtensionDigits = 8`: pinned; matches LatticeLib Code.maxExtensionDigits
- :82 `isWellFormed(_ code:) -> Bool`: 3 ASCII digits + optional `.` + 1–8 ASCII digits; empty/overlong/non-digit extension fails

### Sentence segmentation: Segmenter.swift (extension EideticLib)
- :47 `sentences(_ text:) -> [Substring]`: Apple: NLTokenizer(unit: .sentence); non-Apple (or NLTokenizer yields nothing on non-empty input): falls back to sentencesByDelimiter; empty input → []
- :78 `sentencesByDelimiter(_ text:) -> [Substring]`: canonical cross-platform reference; splits on `.` `!` `?` `\n`, terminator stays attached to preceding segment; trailing remainder is its own segment; total-coverage guaranteed (join(segments) == text) for non-empty input
- Relocated 2026-05-27 (F16) from CorpusKit/Sources/CorpusKit/Chunker.swift::sentenceSegments

## INVARIANTS / GOTCHAS

- fatalError vs UNRESOLVED are TWO DIFFERENT failure modes, do not conflate: FDC.isAvailable==false (missing/broken artifact bundle, a build defect) → fatalError, process terminates. A term with no signature overlap → UNRESOLVED → Anchor(code: "", confidence: 0), a normal, honest, non-fatal result. Never change the UNRESOLVED path to fatalError or vice versa.
- lookup(_:) and lookup(_:recordNovel:) MUST return byte-identical Anchors for the same term; recordNovel only changes whether LatticeLib's shared novel-token pool is written to. Any divergence in the returned Anchor is a bug.
- confidence is a fixed value, not calibrated: EideticLib always emits exactly 0 (UNRESOLVED) or 32 (resolved) from lookup. 16/48/56 are reserved slots in the substrate provenance confidence value set that this library never produces; do not repurpose them here without an explicit design change.
- classifyLatticeCode takes knownCodes as a caller-supplied parameter, not an owned canon: EideticLib deliberately has no opinion on which codes are "known." Do not add a package-level known-code cache; that decision belongs to the caller (federation/version skew is the whole reason pending exists).
- LatticeCodeGrammar.isWellFormed is a SEPARATE implementation from LatticeLib's Code.isWellFormed, not a wrapper. Changing the grammar in one without the other breaks conformance tests silently until those tests run: always update both plus the shared conformance vectors.
- sentences(_:) is NOT cross-platform-identical by design (Apple NLTokenizer vs delimiter fallback): this is the one place in the EideticLib/LatticeLib family where platform divergence is accepted, because downstream consumers content-address chunks by (sourceID, startOffset, text) under an append-only conflict policy rather than requiring byte-identical segmentation. sentencesByDelimiter(_:) is the only segmentation function under cross-leg parity with Rust.
- Total-coverage invariant for both sentence functions: for non-empty input, concatenating the returned segments must reproduce the input exactly, with no gaps, overlaps, or reordering. Any change to either function must preserve this.
- EideticLib.swift and LatticeCodeState.swift and Segmenter.swift do not call one another: there is no internal pipeline. Do not assume ordering or shared state between the three surfaces.
- Package ships an intentionally empty Sources/EideticLib/Resources/ directory (placeholder so `.process("Resources")` has a target); EideticLib bundles zero reference data: do not add data files here expecting lookup to use them. All classification data lives in LatticeLib's bundle.
- Rust Anchor uses explicit `#[serde(rename = "wikidataQID")]`: auto camelCase would produce "wikidataQid" (lowercase id), breaking Swift/Rust JSON interchange. Do not remove the explicit rename.
- Rust LatticeCodeState wire shape is internally tagged: `{"state": "pending", "code": "999.42"}`: locked by `lattice_code_state_wire_shape_conformance` test; must match Swift's Codable encoding.
