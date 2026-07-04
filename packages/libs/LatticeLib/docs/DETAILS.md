---
doc: DETAILS
package: LatticeLib
repo: moot-semantics
authored_commit: bbfef540c1b675b3fb9a493e596499b0fdf2e826
authored_date: 2026-07-04
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
    blob: 6e3cdb0c19c7a7b68992a6bd79780d15f5aa711e
  - path: Sources/LatticeLib/NFKCSubset.swift
    blob: fd2753d05e13b1bc430d2688cc78d0395dca58a6
  - path: Sources/LatticeLib/Normalizer.swift
    blob: f989b93b3dc55751f21bcca65e8e2892bb4193e7
  - path: Sources/LatticeLib/NovelPoolSubmitter.swift
    blob: be91dd71f60a81681c8102567ceeff14ad974526
  - path: Sources/LatticeLib/NovelTokenCache.swift
    blob: 70342e766178eb15c14f52b28cb51d9aa11c6890
  - path: Sources/LatticeLib/NovelTokenTaggerChoice.swift
    blob: 0c9b84a315ca7b31e4bf626f1830c3eea243d8aa
  - path: Sources/LatticeLib/PoolReducer.swift
    blob: e0ccb6a314ec39c47d31dab798841af93c4d394a
  - path: Sources/LatticeLib/QIDClosure.swift
    blob: 11ef3156c12ce27d6b1c1401e3b47a94c3f1bf63
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

# LatticeLib Details

This document walks through every source file in the package. Read
`OVERVIEW.md` first for the big picture. Files appear here in pipeline order:
the module surface, the text primitives, the word-class machinery, the
learning loop, and finally the classification engine and its helpers.

## LatticeLib.swift

This file provides the module surface: the public `LatticeLib` enum and the
module version.

Swift has no module-level functions, so libraries often use an empty enum as
a namespace. The `LatticeLib` enum is that namespace. Other files extend it;
the word-class tagging functions in `WordClassTagger.swift` all live in an
extension of this enum.

`LatticeLib.version` is the module version string. It exists so that
consumers can record which library version produced a result. The version is
bumped in lockstep with the bundled FDC artifacts when new signatures ship,
because a result is only reproducible against the exact artifact set that
produced it.

## Tokenizer.swift

This file provides `Tokenizer`, which splits raw text into words.

Splitting text into words sounds simple but is full of edge cases:
apostrophes, hyphens, non-Latin scripts. Rather than invent rules, the
tokenizer follows Unicode Standard Annex #29, the industry-standard
word-boundary specification. Following a public standard is what allows the
Swift leg and the Rust leg to agree byte for byte: both implement the same
published rules, and a shared conformance fixture proves it.

`Tokenizer.tokenize(_:)` returns the words of a string in input order.
Whitespace and punctuation are dropped. It works by asking Foundation to
enumerate word substrings, which routes to ICU's word-boundary analyzer. The
Rust port uses the `unicode-segmentation` crate, which implements the same
annex.

## NFKCSubset.swift

This file provides the internal compatibility-fold table used by the
normalizer. It maps visually equivalent Unicode characters to plain ASCII: a
full-width `Ａ` becomes `A`, the ligature `ﬁ` becomes `fi`, a superscript `²`
becomes `2`.

Why not use the platform's full Unicode NFKC normalization? Because the Rust
leg cannot reproduce it byte for byte without a large external dependency,
and the two legs must never disagree. Instead, both legs implement the same
explicit table, defined once here and mirrored verbatim in
`rust/src/normalizer.rs`. Same table plus same algorithm equals identical
output by construction. Every mapping is a strict subset of official NFKC,
limited to mappings whose target is plain ASCII, because those are the ones
that improve matching for an ASCII-English corpus.

`fold(_:)` rewrites a string scalar by scalar, replacing each mapped
character and passing everything else through unchanged. `foldScalar(_:)`
holds the actual mapping: an explicit literal table first, then arithmetic
range families (full-width forms, super/subscripts, circled letters, Roman
numerals) expressed as offsets so the table stays small and auditable.
Canonical recomposition (for example, half-width katakana) is deliberately
out of scope; the file documents this so a future reader does not mistake the
omission for a bug.

## Normalizer.swift

This file provides `Normalizer`, the canonicalization front door. It runs
before stemming and lexicon lookup so that different surface spellings of the
same word collapse to one key.

`Normalizer.normalize(_:)` applies the `NFKCSubset` fold and then a
Unicode-aware lowercase. The fold runs first for a reason the code spells
out: a full-width `Ａ` must first become `A` and then fold to `a`, matching
the key that a plain `a` produces. Lowercasing alone would leave the
full-width form distinct, and the same word would split into two different
lexicon keys. The function is deterministic and byte-identical to the Rust
port for every input in the shared conformance fixture.

## Stemmer.swift

This file provides `Stemmer`, a hand-ported implementation of the Porter2
(Snowball English) stemming algorithm. Stemming reduces a word to its root:
"dogs" becomes "dog," "classification" becomes "classif." The point is
recall: text about "running" and text about "runs" should reach the same
lexicon entry.

Porter2 removes English suffixes in five ordered steps, guided by two regions
of the word (R1 and R2) computed from vowel-consonant transitions. Most
removals only apply when the suffix sits inside a region, which prevents
overstripping short words. Two exception lists handle irregular forms, such
as "skies" becoming "sky" rather than "ski."

`Stemmer.stem(_:)` runs the full algorithm: preprocessing (apostrophe and
consonant-y marking), region computation, then steps 0 through 5. Words
shorter than three letters return unchanged, per the specification.
`Stemmer.bundledReferenceCorpus()` exposes the pinned test corpus
(`Resources/SnowballEnglish.json`) so the conformance test can verify that
this port and the Rust port (which uses the `rust-stemmers` crate) produce
byte-identical stems for every input.

## WordClass.swift

This file provides `WordClass`, the label produced by tagging Step 1 of the
encoder: `.noun`, `.verb`, or `.other`.

Step 1 keeps only nouns and verbs because they carry the subject matter of a
sentence. `.other` is the discard bucket for everything else — articles,
prepositions, adjectives, punctuation. The enum is backed by strings so it
serializes to a stable, human-readable JSON form (`"noun"`, `"verb"`,
`"other"`) that the shared conformance vectors and the Rust port both read.

## WordClassTable.swift

This file provides the noun/verb fast-path table and its live-swappable
process cache. The table is the first stop for every token: a token present
in it resolves to its word class in constant time, with no tagger invoked.

Three load functions cover the table's life cycle. `loadBundled()` reads the
pinned snapshot shipped in `Resources/WordClassTable.json`.
`loadWritable()` reads the merged artifact that `PoolReducer` maintains
outside the app bundle. `loadWithPrecedence()` is the canonical entry: it
prefers the writable merged artifact, because that one contains learned
tokens, and falls back to the bundled pristine table.

`WordClassTableCache` solves a harder problem: adopting a new table while the
process is running. After the reducer merges new tokens, waiting for a
process restart would delay learning indefinitely. The cache instead
publishes an immutable snapshot (`WordClassTableSnapshot`, the parsed table
plus precomputed membership sets) behind a lock. Readers copy the snapshot
reference out and test membership outside the lock, so a swap can never
produce a torn read: a reader sees the whole old table or the whole new one,
never a mix. `swap(_:)` publishes a new snapshot and bumps a version counter.
`reloadFromPrecedence()` and `reload(fromArtifact:)` re-resolve the table
from disk and publish it. The version counter matters for determinism:
tagging is deterministic given (input, table version), and the counter makes
the version observable.

## WordClassTagger.swift

This file provides the public tagging API, `LatticeLib.wordClass`, in its
four overloads. This is encoder Step 1 as callers see it.

Every overload shares the same fast path. The token is lowercased, then
checked against the live table snapshot: the verb set first, then the noun
set, so a token listed in both resolves to verb. Only a token absent from the
table — a novel token — reaches a tagger.

`wordClass(_:)` is the default path. Novel tokens go to the deterministic
HMM tagger on all platforms, including Apple. This is the rule that keeps
the Swift and Rust ports bit-identical: Apple's NLTagger is a black box that
can change between OS releases, so it is never in the default path.

`wordClass(_:tagger:)` threads an explicit estate choice. An estate is one
user's memory store; each estate picks its novel-token tagger once, at
creation. The `.hmm` choice is the cross-platform baseline. The `.nlTagger`
choice opts in to Apple's NaturalLanguage tagger for higher accuracy at the
cost of cross-platform determinism, and it is gated by
`taggerEnabled(osVersion:minOSVersion:)`: the table pins the NLTagger OS
version that seeded it, and an older OS fails closed to `.other` rather than
invoke a differently-behaving tagger.

`wordClass(_:recordNovel:)` and `wordClass(_:tagger:recordNovel:)` add
privacy control. By default a novel token's tag is recorded into the shared
pool cache so the table can learn it later. When the text being classified is
private memory content, callers pass `recordNovel: false` and the side effect
is suppressed. The returned tag is identical either way; only the recording
changes. The shared cache itself (`sharedNovelCache`) is wired to a real file
writer only when the `LATTICE_POOL_DIR` environment variable is set;
otherwise the submitter is a no-op, so no deployment leaks tokens to disk by
default.

## HMMTagger.swift

This file provides the deterministic novel-token tagger: a three-state Hidden
Markov Model (noun, verb, other) decoded with integer Viterbi.

Why a statistical model here at all? Because novel tokens are exactly the
words no table covers, so some generalization is required. And why integer
arithmetic? Because floating-point rounding can differ across platforms and
compilers, and this tagger must produce the same answer on every port. The
model weights are log-likelihoods scaled by 1,000 and rounded to integers at
build time; runtime scoring is pure integer addition and comparison.

The observation alphabet is morphological: each token maps to one observation
such as "ends in -ing," "ends in -tion," "contains a non-letter," or "plain."
`observe(_:)` checks these in a fixed priority order that both ports
replicate exactly. `tag(_:)` then picks the state with the best initial plus
emission weight; for a one-token sequence, Viterbi reduces to that argmax,
with ties resolved to the lowest state index.

The weights ship in the frozen artifact `Resources/HMMTaggerModel.json`,
trained on the MASC 3.0.0 Penn Treebank annotation. The training deliberately
uses only the corpus's rare words (frequency 1). The tagger only ever sees
out-of-vocabulary words, and rare words are the best statistical proxy for
those, which gives the correct noun-leaning prior for unknown words.
Estimating from the full corpus would let frequent function words dominate
and wrongly default unknowns to `.other`.

## NovelTokenTaggerChoice.swift

This file provides `NovelTokenTaggerChoice`, the two-case enum (`.hmm`,
`.nlTagger`) that selects the novel-token tagger for an estate.

The type mirrors an identically named enum in PersistenceKit, where the
choice is stored on the estate configuration. The two packages define it
independently so that neither depends on the other; a consumer bridges
between them with a trivial switch. The file documents the threading rule:
pass the choice down as a parameter from the estate configuration to the
call sites, never through global state. `.hmm` is the default and the
federation-safe choice; `.nlTagger` is Apple-only and trades determinism for
accuracy.

## NovelTokenCache.swift

This file provides the local accumulation cache for novel-token tags, plus
the two wire-format types `PoolEntry` and `PoolSubmission`.

When the tagger classifies a novel token, the result is worth keeping: if
many devices see the same unknown word, the shared table should learn it.
`NovelTokenCache.record(token:wordClass:)` appends the tagged token to a
pending list under a lock. At exactly `poolSubmitThreshold` (50) entries, the
cache builds a `PoolSubmission` — stamped with the table version, platform,
and tagger version — drains the list, and hands the payload to an injected
submitter closure outside the lock. Entries below the threshold are kept
indefinitely; there is no aging or cleanup, by contract.

The submitter is injected at construction so tests can assert the drain
without touching the file system, and so hosts that never configure a pool
get a no-op. The version stamps exist because the pool consumer discards
submissions made against a stale table version; an observation is only
meaningful relative to the table that failed to contain it.

## NovelPoolSubmitter.swift

This file provides the production submitter factory: it turns a drained
`PoolSubmission` into a JSON file in a local pool directory.

The pool is a directory of files rather than a database or a network call.
Files are cheap, observable, and easy to reason about; a future reducer can
process them in name order because each file name embeds an ISO 8601
timestamp plus a short unique suffix. Writes are atomic and fire-and-forget:
a failed write is logged and the data is discarded for that cycle, never
retried and never crashed on, because pool data is recoverable from future
observations.

`make(poolDirectory:)` returns a submitter closure bound to a directory.
`makeDefault()` binds to the resolved default. `poolDirectory()` resolves
that default: the `LATTICE_POOL_DIR` environment variable when set, otherwise
an Application Support path on Apple platforms or an XDG data path elsewhere.
It is public because the write side (submitter) and the read side (the
governor that triggers the reducer) must agree on the path, so it lives here
as the single source of truth. `tableArtifactURL()` resolves the sibling
location of the writable merged table — `WordClassTable.json` next to the
pool directory — which is the file the reducer updates and
`WordClassTable.loadWritable()` reads.

## PoolReducer.swift

This file provides the batch job that closes the learning loop: it reads
pooled submissions and merges qualifying novel tokens into the writable
word-class table artifact.

`PoolReducer.reduce(poolDirectory:tableArtifactURL:now:maxFiles:)` runs in
steps. It first seeds the writable artifact from the bundled table if none
exists, because the reducer cannot write into the read-only app bundle. It
then loads the table, enumerates `pool_*.json` files, and processes the
oldest `maxFiles` of them; bounding the batch keeps a large backlog from
stalling the caller, and the remainder drains on later runs. Each file is
decoded and version-checked: a malformed or stale-version file is moved to a
quarantine directory rather than deleted, preserving forensic value. Within
the run, the first occurrence of each token wins, tokens already in the table
are skipped, and only NOUN and VERB tags expand the table — an OTHER tag is
already represented by absence. Consumed files move to an archive directory,
which makes the reducer idempotent: re-running on a drained pool is a no-op.
If anything was consumed, the artifact is rewritten atomically with sorted
keys and an advanced snapshot date.

`PoolReduceResult` summarizes a run (consumed, quarantined, added, skipped),
and `PoolReducerError` covers the three failure classes (table read, table
write, pool directory unreadable). The file warns against wiring the reducer
into the per-token tagging path: reduction is a batch operation, triggered by
a resident governor on an idle cadence or by an operator command.

## Lexicon.swift

This file provides the canonicalization lexicon type and its deterministic
builder. The lexicon is encoder Step 2's reference data: a flat map from
`stem(normalize(token))` to a concept identity.

`CanonicalizationLexicon` is the pinned artifact shape: a version, a
language, and the entries map. The version is part of the agreement protocol:
two encoders must share it, or their concept bags can diverge.

`LexiconBuilder.build(_:)` constructs the artifact from two public sources.
Wikidata supplies the concept identities: surface forms and aliases mapped to
Q-IDs, plus the property (P8814) that links WordNet synsets to Q-IDs.
WordNet supplies word senses in frequency order. The build considers every
candidate key with a three-tier rule, lower tier winning: first a Q-ID
reached through a WordNet sense of the word (this is where WordNet
disambiguates — "dog" maps to the animal, not the sausage, because the animal
is the primary sense); then a Q-ID known only from a Wikidata alias; and
last a `wn:` synset fallback, used only when no Q-ID exists for the concept.
Within each tier, explicit deterministic tie-breaks (sense rank, support
count, lowest Q-number) make the result order-independent: same inputs, same
artifact, byte for byte, on any machine. Keys are derived with the same
normalize-then-stem pipeline the runtime uses, so build-time and runtime keys
agree exactly. Only single-token surfaces become keys, because the runtime
looks up one token at a time.

## ConceptBag.swift

This file provides encoder Steps 1 through 3: `BagBuilder` turns a block of
text into a concept bag, the weighted map from concept identity to
occurrence count.

`bag(_:lexicon:keep:)` makes one pass over the tokens. Each token is
normalized and stemmed, then looked up in the lexicon. The token is kept if
its word class is in the keep set (nouns and verbs by default) — or if it
resolves to a Wikidata Q-ID. That second condition is a deliberate
relaxation: part-of-speech taggers often mislabel named entities, and a
proper noun like "Everest" matters more to classification than most common
nouns. Deciding membership from the pinned lexicon keeps the relaxation
deterministic, because the lexicon is identical everywhere, while
cross-platform proper-noun tagging is not. A kept token contributes its
concept identity when the lexicon has one, or its bare stemmed surface form
otherwise; a surface form can still match a signature carrying the same
string.

`bag(_:lexicon:keep:recordNovel:)` is the privacy variant. It produces a
byte-identical bag, but when `recordNovel` is `false`, novel tokens
encountered during tagging are not accumulated into the shared pool cache.
This is the seam the memory-capture path uses so that private text never
reaches the pool pipeline. `bag(_:lexicon:keep:taggerChoice:)` is the
estate-configuration variant: it threads the estate's explicit tagger choice
into Step 1 so the bag is built consistently with the estate's other
indexed content.

## Code.swift

This file provides the FDC code grammar: what makes a code string
well-formed.

A code is a three-digit integer (000 through 999) with an optional decimal
extension of one to eight digits, such as `540` or `540.137`. Each extension
digit subdivides its parent into ten, so the extension carries leaf
resolution under a spine code. The eight-digit cap bounds the printable form
at twelve characters so tools can reason about column widths and string
lengths.

`Code.isWellFormed(_:)` checks the grammar and nothing else. Validity is
purely grammatical by design: a well-formed code is accepted, stored, and
round-tripped even if no term currently encodes to it, because whether a code
is "known" belongs to the caller's own known-code set, not to the grammar.
`Code.integerBase(of:)` extracts the three-digit integer base of a
well-formed code, or `nil` for a malformed one.

## FDCFrame.swift

This file provides the frame model — the versioned list of codes and labels —
and the ancestry math over code strings.

`FDCEntry` is one code with its heading label, preserved verbatim including
subject markers, because build tooling consumes the raw heading text.
`FDCFrame` is the versioned list. Ancestry is not stored anywhere; it is
derived from the code string itself.

The derivation has two regimes, and the file's long comment explains why a
naive dot-split is wrong. The three-digit integer head is a Dewey-style
positional hierarchy read at the hundreds, tens, and units places: the parent
of `006` is `000`, and the parent of `510` is `500`. The decimal tail is a
per-segment hierarchy: the parent of `006.6` is `006`. `decimalParent(of:)`
implements both regimes as a pure function of the string. `ancestors(of:)`
walks parents upward and returns the chain root-first, so
`ancestors("006.6")` is `["000", "006"]`. `children(of:)` filters the frame's
codes by parent identity and sorts the result for deterministic output.

## CodeSignature.swift

This file provides the build-time signature assembly: how a code's
characteristic term set is put together before it ships as an artifact.

Every code in the frame has three source texts: its label, the title of its
reference article, and the article body. `SourceWeights` pins their relative
importance — label 3, title 2, article 1 — because the label is the most
precise statement of what a code means and the article body the broadest.
`SignatureAssembler.merge(label:title:article:weights:)` combines the three
concept bags into one weighted term map.

`SignatureAssembler.accumulateAncestors(ownTerms:ancestorsOf:)` then adds
every ancestor's own terms into each code's signature. A specific code such
as `540.1` thereby carries the full vocabulary of its lineage, so text that
talks about the general subject still supports the specific code during
descent. The result is a `CodeSignature` per code; the `fingerprint` field
stays `nil` until a later SimHash pass computes it, once the global concept
vocabulary is known. All of this is build-time machinery: the runtime ships
only the compact term sets.

## FDCMatcher.swift

This file provides encoder Steps 4 and 5: scoring a concept bag against the
signatures and descending the frame to the most specific supported code. It
is the heart of the classifier.

`FDCMatcher.init(lexicon:frame:signatures:stopThreshold:scoreMode:)` builds
the runtime index once. Every term across all signatures is interned: sorted
alphabetically and assigned a dense integer identifier. The hot path then
works entirely with integer keys, which profiling showed removes the dominant
cost (string hashing) on large imports. The identifier order is deliberately
the alphabetical order, so sorting integer sets visits terms in the same
sequence as sorting the original strings — floating-point sums are computed
in the same order and stay bit-identical to the pre-interning implementation.
The initializer also builds an inverted index (term to codes), document
frequencies for IDF weighting, and per-signature norms.

`ScoreMode` selects how overlap becomes a score. `.raw` sums the bag counts
of shared terms. `.idf` weights each shared term by its rarity across
signatures (inverse document frequency), so distinctive vocabulary counts for
more than vocabulary that appears everywhere. `.cosine` and `.idfCosine`
additionally divide by a signature-size norm to remove the advantage of big
signatures. The shipped runtime uses `.idf`, which measurably improved code
selection over `.raw` on the v1.0 frame.

`encode(_:)` and the two `encodeAnchor` overloads are the public entries; the
`recordNovel: false` variant suppresses pool accumulation for private text.
All delegate to one private method, `encodeFromBag(_:)`, which runs the
algorithm. Step 4 collects candidate codes from the inverted index, scores
each, and takes the best, breaking ties toward the lowest code in a fixed
scan order so the result never depends on hash ordering. Two guards protect
honesty: an empty or non-matching bag returns `nil` (UNRESOLVED, never a
guess), and when more than `maximumTiedWinnersForClassification` (4) codes
tie at the top score, the matcher also returns UNRESOLVED, because a wide tie
means the text's vocabulary is generic and any single winner would be
confidently wrong. Step 5 then walks down the frame: a child must carry at
least `stopThreshold` raw overlap to be a candidate (the cutoff is
mode-independent by design), the best-scoring child wins, and the walk
repeats until no child qualifies. The deepest reached code is the answer.

`dominantQID(_:)` computes the other half of the anchor: the
highest-counted Wikidata Q-ID in the bag, ties broken toward the lowest
Q-ID, or `nil` when the bag has none. This is "what the text is most about,"
surfaced in the same pass so consumers fill an anchor without re-encoding.

## FDCRuntime.swift

This file provides `FDC`, the runtime facade that consumers actually call.
It owns the pinned artifacts and hides the matcher behind four static
functions.

The three runtime artifacts — lexicon, frame, and compact signatures — load
once per process from the module bundle, together, or not at all. The
signatures loaded here are the compact form (code to term list) because the
runtime matcher uses only term membership; the weighted form remains a build
record. The loaded matcher is configured with `scoreMode: .idf` and the
pinned `stopThreshold` of 1 — a value that testing showed is inert on the
shallow v1.0 frame, pinned for the contract rather than for tuning.

`FDC.encode(_:)` returns the lattice code or `nil` for UNRESOLVED.
`FDC.encodeAnchor(_:)` adds the dominant concept Q-ID; the
`recordNovel: false` overload is the privacy seam for memory capture.
`FDC.isAvailable` reports whether the artifacts loaded, and
`FDC.dataVersion` reports the signatures version so callers can record
provenance with every classification. `FDC.ancestors(of:)` exposes the
frame's ancestry walk so consumers get the chain without reaching into
`FDCFrame` directly. `FDC.label(for:)` returns a human-readable heading for
a code; for three-digit codes it walks up one level first, because leaf
labels at that depth are often multi-subject compounds while the parent
carries the cleaner single-topic heading that dashboards want.

## QIDClosure.swift

This file provides the Q-ID ancestor surface: given a Wikidata Q-ID, it
returns every ancestor reachable through instance-of (P31) and subclass-of
(P279) relations.

Consumers use this to generalize a concept — from "dog" up through "mammal"
and "animal" — without any network access. The edge graph is a pinned,
offline Wikidata snapshot (`Resources/QIDClosureEdges.json`), produced by
build tooling and checked in like the other artifacts. The runtime never
queries Wikidata.

`QIDClosure.ancestors(of:)` computes the transitive closure with an
iterative breadth-first walk over the edge map, excluding the queried Q-ID
itself, and returns the result sorted numerically by Q-number. The walk
dedupes visited nodes, so even a cycle in the source data would terminate.
Results are memoized per Q-ID for the life of the process; the memo is
guarded by a lock, while the walk itself runs outside the lock, and a race
between two threads computing the same Q-ID is harmless because both produce
the identical pure result. `isAvailable` and `dataVersion` mirror the
`FDC` facade: artifact-presence and provenance.

## LexRank.swift

This file provides the article-reduction step used at build time: LexRank,
which selects the most central sentences of a long article before the
encoder runs over it.

Code signatures are built from reference articles, but a whole article is
noisy; the useful vocabulary is concentrated in its most representative
sentences. LexRank finds them with eigenvector centrality — the idea behind
PageRank — over a sentence-similarity graph: sentences are nodes, and two
sentences connect when their stemmed term vectors are cosine-similar above a
threshold (0.1). Sentences that many other sentences resemble score high.

`LexRank.reduce(_:sentences:)` segments the text, builds per-sentence
term-frequency vectors with the shared primitives, assembles the thresholded
similarity graph, and asks `SubstrateML.EigenvalueCentrality` for scores. It
returns the top N sentences (default 10) joined in their original order, or
the text unchanged when it is already short enough. This is build-time-only
machinery: it never runs on user devices, so it has no cross-platform
agreement constraint, though it is still deterministic. It is also the one
place the package uses its single dependency, SubstrateML.

## Rust Port and Conformance

The `rust/` directory contains the second leg of the library: seventeen
source files mirroring the Swift implementations, from `tokenizer.rs` through
`fdc_runtime.rs`. The two legs share their pinned artifacts and are gated by
shared conformance fixtures in `rust/tests/fixtures/` — recorded
input-output pairs for normalization, tagging, and full FDC encoding that
both implementations must reproduce byte for byte. When you change either
leg, run both test suites; the fixtures are the contract.
