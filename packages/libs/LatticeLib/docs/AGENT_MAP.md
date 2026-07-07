---
doc: AGENT_MAP
package: LatticeLib
repo: moot-semantics
authored_commit: ee425fbef9955ae233794d035902d12db4348044
authored_date: 2026-07-07
sources:
  - path: Sources/LatticeLib/Code.swift
    blob: e2c8307da3a2821bbabd7055a8f714754289df0f
  - path: Sources/LatticeLib/CodeSignature.swift
    blob: b9245f8e8a8c913bcdcb27287aab45cd2729abd9
  - path: Sources/LatticeLib/ConceptBag.swift
    blob: db18742c50e495682521bd7dd544b3725969b467
  - path: Sources/LatticeLib/FDCFrame.swift
    blob: 8d83ab902206ffdb4d3ee9351a6443c90bcead77
  - path: Sources/LatticeLib/FDCMatcher.swift
    blob: 330b99e7af1d6366425d4956652a3dd11b1a1c4b
  - path: Sources/LatticeLib/FDCRuntime.swift
    blob: 2fd34b7657f6888117fb9d4226ef314769d75800
  - path: Sources/LatticeLib/HMMTagger.swift
    blob: 45ecd95be83f7416851a80070f85aa05aea7fc4a
  - path: Sources/LatticeLib/LatticeLib.swift
    blob: 88712d0a73a75d9400741a871719f39be2edc584
  - path: Sources/LatticeLib/Lexicon.swift
    blob: 2a51dfbf417b542b60f5b5597e0850957c5b6631
  - path: Sources/LatticeLib/LexRank.swift
    blob: 5c6fb919539aa3b59e71ff8d351cbfa71a2d7a9e
  - path: Sources/LatticeLib/NFKCSubset.swift
    blob: fd2753d05e13b1bc430d2688cc78d0395dca58a6
  - path: Sources/LatticeLib/Normalizer.swift
    blob: f989b93b3dc55751f21bcca65e8e2892bb4193e7
  - path: Sources/LatticeLib/NovelPoolSubmitter.swift
    blob: e939e0b8858a2b6bec2cc6daf6196268ecf0d9e8
  - path: Sources/LatticeLib/NovelTokenCache.swift
    blob: 70342e766178eb15c14f52b28cb51d9aa11c6890
  - path: Sources/LatticeLib/NovelTokenTaggerChoice.swift
    blob: 0c9b84a315ca7b31e4bf626f1830c3eea243d8aa
  - path: Sources/LatticeLib/PoolReducer.swift
    blob: e0ccb6a314ec39c47d31dab798841af93c4d394a
  - path: Sources/LatticeLib/QIDClosure.swift
    blob: bffb053a66467662b967b0d2d8619e047e260ecf
  - path: Sources/LatticeLib/Stemmer.swift
    blob: 042fe54236cbe61f1cb4126dc27dde97a3f5cc5d
  - path: Sources/LatticeLib/Tokenizer.swift
    blob: 82a8b85b5c31163d51194c7f28320305f8c798af
  - path: Sources/LatticeLib/WordClass.swift
    blob: 9d4851967dca01b266ad0d8f4f35cf4a757d3d7d
  - path: Sources/LatticeLib/WordClassTable.swift
    blob: 731f6f90c59aae71c73114ede301877d1bf8035f
  - path: Sources/LatticeLib/WordClassTagger.swift
    blob: 15e27da5cdeca3540654914992e8c78f83ed937a
---

# AGENT_MAP: LatticeLib

PURPOSE: deterministic on-device text→classification-code engine (FDC: Frame-Directed Classification) + shared text primitives. Text → concept bag → scored match against pinned code signatures → decimal-frame descent → code (or nil=UNRESOLVED).

DEPS: imports SubstrateML (EigenvalueCentrality, build-time LexRank only), NaturalLanguage (LexRank sentence-split; optional NLTagger path), OSLog. Imported by: EideticLib, AriaMcpKit, apps/moot-mgr, tools/seed-generator. Rust port in rust/ mirrors everything; conformance fixtures rust/tests/fixtures/*.json gate byte-identity.


CURRENT TRUE-UP:
- v1.0.24: LexRank passes explicit offline telemetry sentinels. NovelPoolSubmitter caps pool files at 500. QIDClosure skips self-cycles.

ENTRY POINTS (most callers need only these):
- FDCRuntime.swift:32 `FDC.encodeAnchor(_ text:) -> (code: String?, conceptQID: String?)`: main runtime classify
- FDCRuntime.swift:45 `FDC.encodeAnchor(_:recordNovel:)`: same, recordNovel:false = privacy seam (no pool writes)
- WordClassTagger.swift:65 `LatticeLib.wordClass(_ token:) -> WordClass`: single-token noun/verb/other

## Symbol Table

### Runtime facade: FDCRuntime.swift
- :15 `enum FDC`: artifact-owning facade; loads lexicon+frame+signatures once/process
- :23 `FDC.stopThreshold = 1`: pinned descent cutoff (inert on shallow v1.0 frame)
- :27 `encode(_ text:) -> String?`: code only; nil = UNRESOLVED or artifacts missing
- :32/:45 `encodeAnchor(...)`: code + dominant Q-ID; see ENTRY POINTS
- :50 `isAvailable: Bool`: artifacts loaded?
- :54 `dataVersion: String`: signatures version for provenance; "0.0.0-unavailable" on load failure
- :67 `ancestors(of code:) -> [String]`: frame ancestry, root-first, excl. self
- :84 `label(for code:) -> String?`: human heading; 3-digit codes report parent's label (leaf labels are multi-subject compounds)

### Matcher: FDCMatcher.swift
- :30 `struct FDCMatcher: Sendable`: Steps 4–5; direct init for custom artifacts/tests
- :49 `enum ScoreMode`: .raw/.idf/.cosine/.idfCosine; runtime ships .idf, direct-init default .raw
- :63 `stopThreshold: Int`: raw-overlap descent cutoff, mode-independent
- :120 `init(lexicon:frame:signatures:stopThreshold:scoreMode:)`: interns terms→dense Int ids (alphabetical order ⇒ Int sort == String sort ⇒ bit-stable float sums); builds inverted index + idf + norms
- :267 `maximumTiedWinnersForClassification = 4`: >4 tied argmax winners ⇒ UNRESOLVED (generic-vocabulary guard)
- :271 `encode(_:)`, :280 `encodeAnchor(_:)`, :296 `encodeAnchor(_:recordNovel:)`: public encode surface

### Bag construction: ConceptBag.swift
- :25 `typealias ConceptBag = [String: Int]`: conceptID|surface → count
- :27 `enum BagBuilder`: Steps 1–3
- :37 `bag(_:lexicon:keep:)`: default (keep = [.noun,.verb]); Q-ID lexicon hit bypasses POS filter (named-entity relaxation, deterministic via pinned lexicon)
- :82 `bag(_:lexicon:keep:recordNovel:)`: identical bag; recordNovel:false suppresses pool side effect
- :118 `bag(_:lexicon:keep:taggerChoice:)`: estate-configured tagger threading

### Text primitives
- Tokenizer.swift:21 `Tokenizer.tokenize(_:) -> [String]`: UAX #29 words, input order
- Normalizer.swift:55 `Normalizer.normalize(_:) -> String`: NFKCSubset.fold then lowercased; fold MUST run first
- NFKCSubset.swift:40 `NFKCSubset.fold(_:)` (internal): ASCII-target compatibility subset; table mirrored verbatim in rust/src/normalizer.rs: edit both or neither
- Stemmer.swift:69 `Stemmer.stem(_:) -> String`: Porter2/Snowball English; <3 chars unchanged
- Stemmer.swift:55 `bundledReferenceCorpus() -> Data?`: conformance corpus accessor

### Word-class tagging: WordClassTagger.swift (extension LatticeLib)
- :65 `wordClass(_:)`: table fast path (verb set checked BEFORE noun set) → HMM fallback; NEVER NLTagger
- :102 `taggerEnabled(osVersion:minOSVersion:)`: NLTagger OS gate; unparseable minOS fails closed
- :208 `wordClass(_:recordNovel:)`: pool-suppressing variant
- :285 `wordClass(_:tagger:)` / :330 `wordClass(_:tagger:recordNovel:)`: estate-choice dispatch; .nlTagger fails closed to .other below min OS, falls back to HMM on non-Apple
- WordClass.swift:23 `enum WordClass: String`: noun|verb|other; string-backed for wire stability
- NovelTokenTaggerChoice.swift:22 `enum NovelTokenTaggerChoice`: .hmm (default, federatable) | .nlTagger (Apple-only, non-deterministic cross-platform); mirrors PersistenceKit type, bridge by switch
- HMMTagger.swift:70 `enum HMMTagger` (internal): 3-state integer Viterbi; artifact HMMTaggerModel.json (hmm-viterbi-3, MASC 3.0.0 rare-word-trained); tie → lowest state index (noun<verb<other)

### Word-class table: WordClassTable.swift
- :39 `struct WordClassTable`: table_version/min_os_version/snapshot_date/nouns/verbs
- :94 `loadBundled()` / :118 `loadWritable()` / :145 `loadWithPrecedence()`: precedence: writable merged artifact first, bundled fallback; use loadWithPrecedence
- :202 `enum WordClassTableCache`: process-wide live-swappable snapshot holder
- :243 `version: UInt64`: bumps per swap; determinism keyed on (input, table-version)
- :255 `swap(_:)` / :268 `reloadFromPrecedence()` / :283 `reload(fromArtifact:)`: atomic snapshot publish, no torn reads

### Novel-token pool loop
- NovelTokenCache.swift:86 `final class NovelTokenCache`: locked pending list; drains at exactly :91 `poolSubmitThreshold = 50`; submitter injected, called outside lock
- NovelTokenCache.swift:30 `PoolEntry` / :42 `PoolSubmission`: wire format (snake_case keys; tags "NOUN"/"VERB"/"OTHER")
- NovelPoolSubmitter.swift:49 `make(poolDirectory:)` / :63 `makeDefault()`: file-writing submitter factory; fire-and-forget, no retry
- NovelPoolSubmitter.swift:81 `poolDirectory()`: LATTICE_POOL_DIR > Apple App Support > XDG; single source of truth for read+write sides
- NovelPoolSubmitter.swift:97 `tableArtifactURL()`: writable merged WordClassTable.json, SIBLING of pool dir
- PoolReducer.swift:154 `PoolReducer.reduce(poolDirectory:tableArtifactURL:now:maxFiles:)`: batch merge; seeds writable artifact from bundle; quarantines malformed/stale-version files; NOUN/VERB only; first-occurrence dedup; archives consumed files (idempotent); DO NOT call from hot path
- PoolReducer.swift:65 `PoolReduceResult` / :105 `PoolReducerError`: result summary / failure classes

### Frame + codes
- Code.swift:34 `Code.isWellFormed(_:)`: grammar only: 3 digits + optional ≤8-digit decimal ext (:31 maxExtensionDigits=8); known-vs-pending is caller's concern
- Code.swift:58 `Code.integerBase(of:)`: "540.137" → 540
- FDCFrame.swift:17 `FDCEntry`: code + verbatim label
- FDCFrame.swift:34 `FDCFrame`: versioned code list; ancestry DERIVED, not stored
- FDCFrame.swift:142 `ancestors(of:)`: TWO regimes: integer head is Dewey positional (parent("006")="000"), decimal tail is per-segment (parent("006.6")="006"); NOT a dot-split
- FDCFrame.swift:131 `children(of:)`: immediate children, sorted

### Build-time only (never on-device)
- Lexicon.swift:30 `CanonicalizationLexicon`: version/language/entries (stem→conceptID); version is part of agreement protocol
- Lexicon.swift:91 `LexiconBuilder.build(_:)`: Wikidata-primary, WordNet-disambiguated; 3-tier deterministic conflict resolution; single-token keys only
- CodeSignature.swift:36 `SignatureAssembler.merge(label:title:article:weights:)`: source weights label 3 / title 2 / article 1 (:13 SourceWeights)
- CodeSignature.swift:52 `accumulateAncestors(ownTerms:ancestorsOf:)`: signature += all ancestors' own terms
- CodeSignature.swift:24 `CodeSignature`: terms + fingerprint (nil until SimHash pass)
- LexRank.swift:25 `LexRank.reduce(_:sentences:)`: top-N central sentences (default :20 =10, cosine threshold :21 =0.1) via SubstrateML.EigenvalueCentrality; deterministic but NOT cross-platform-gated

### Q-ID closure: QIDClosure.swift
- :45 `QIDClosure.ancestors(of qid:) -> [String]`: transitive P31/P279 closure over pinned QIDClosureEdges.json; BFS, excl. self, sorted by Q-number; memoized per process
- :68 `isAvailable` / :73 `dataVersion`: artifact presence / provenance

### Module: LatticeLib.swift
- :13 `enum LatticeLib`: namespace; :17 `version = "1.0.0"` bumped with artifact ships

## INVARIANTS / GOTCHAS

- DETERMINISM IS THE CONTRACT. encode() is pure over (text, artifact versions). Any change to Tokenizer/Normalizer/NFKCSubset/Stemmer/HMMTagger/FDCMatcher scoring must be mirrored in rust/src/ and pass rust/tests/fixtures/{normalize,tag,fdc}_conformance.json + Swift conformance tests. Same for artifact regeneration.
- nil code = UNRESOLVED, by design: empty bag, no signature overlap, or >4 tied winners. Never "fix" by guessing.
- Verb set checked before noun set in every wordClass overload: a token in both is a verb. Do not reorder.
- HMM (not NLTagger) is the default novel-token tagger on ALL platforms incl. Apple. NLTagger only via explicit .nlTagger estate choice; it breaks cross-platform determinism and federation.
- NFKCSubset table + Rust normalizer.rs are ONE table in two files. Never edit one alone.
- FDCMatcher term interning: IDs assigned in ascending String order: required so sorted-Int iteration == sorted-String iteration ⇒ bit-identical float sums. Do not change assignment order.
- FDCFrame ancestry: integer head ≠ decimal tail regimes. parent("006") is "000" (Dewey positional), not "00". A dot-split reimplementation is wrong.
- stopThreshold compares RAW integer overlap, never the (possibly normalized) score.
- Pinned constants: do not change without a new artifact version + conformance regen: poolSubmitThreshold 50, stopThreshold 1, maxExtensionDigits 8, maximumTiedWinnersForClassification 4, SourceWeights 3/2/1, HMM logScale 1000, LexRank 10/0.1.
- Privacy seams: recordNovel:false variants produce byte-identical results with pool accumulation suppressed: required for user memory content (GLK capture, EideticLib.lookup). sharedNovelCache writes files ONLY when LATTICE_POOL_DIR is set; default submitter is a no-op.
- Bundled WordClassTable.json is a small seed (21 nouns / 18 verbs, v1.0.0): in practice most tokens are novel and route through the HMM until the pool loop grows the writable table.
- Artifacts load once per process (static let); FDC/QIDClosure fail soft (isAvailable=false, encode→nil): check isAvailable in hosts with unusual bundle setups.
- PoolReducer is batch-only (governor idle-tick or operator CLI). Bounded by maxFiles per run; backlog drains across runs.
- WordClassTableCache.swap is the ONLY legal live-update path for tagging behavior; readers get whole-snapshot consistency, and version() observability keys determinism.
- LexRank is build-time-only; it may use floats and NLTokenizer freely. Do not move it into the runtime encode path.
