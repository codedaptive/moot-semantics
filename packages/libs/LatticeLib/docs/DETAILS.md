---
doc: DETAILS
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

# LatticeLib Details

## Current Release Details

NovelPoolSubmitter drops new submissions when the pool directory already has 500 files.
Each file holds one drain cycle.
QIDClosure now ignores a cyclic edge back to the queried Q-ID.
LexRank passes `estate: ""` and `ts: 0`.
Its build-time article work has no estate scope.

This document walks through every source file in the package. Read
`OVERVIEW.md` first for the big picture. Files appear here in pipeline
order. First comes the module surface. Then come the text primitives, the
word-class machinery, and the learning loop. Last comes the classification
engine and its helpers.

## LatticeLib.swift

This file provides the module surface. It holds the public `LatticeLib`
enum and the module version.

Swift has no module-level functions. So libraries often use an empty enum
as a namespace. The `LatticeLib` enum is that namespace. Other files
extend it. The word-class tagging functions in `WordClassTagger.swift` all
live in an extension of this enum.

`LatticeLib.version` is the module version string. It lets consumers
record which library version produced a result. The version is bumped in
lockstep with the bundled FDC artifacts when new signatures ship. A result
is only reproducible against the exact artifact set that produced it.

## Tokenizer.swift

This file provides `Tokenizer`, which splits raw text into words.

Splitting text into words sounds simple. It is full of edge cases instead:
apostrophes, hyphens, non-Latin scripts. The tokenizer does not invent its
own rules. It follows Unicode Standard Annex #29, the industry-standard
word-boundary specification. This public standard is what lets the Swift
leg and the Rust leg agree byte for byte. Both implement the same
published rules. A shared conformance fixture proves it.

`Tokenizer.tokenize(_:)` returns the words of a string in input order. It
drops whitespace and punctuation. It works by asking Foundation to
enumerate word substrings. This routes to ICU's word-boundary analyzer.
The Rust port uses the `unicode-segmentation` crate. That crate implements
the same annex.

## NFKCSubset.swift

This file provides the internal compatibility-fold table used by the
normalizer. It maps visually equivalent Unicode characters to plain ASCII.
A full-width `Ａ` becomes `A`. The ligature `ﬁ` becomes `fi`. A superscript
`²` becomes `2`.

Why not use the platform's full Unicode NFKC normalization? The Rust leg
cannot reproduce it byte for byte without a large external dependency.
The two legs must never disagree. So both legs instead implement the same
explicit table. It is defined once here and mirrored verbatim in
`rust/src/normalizer.rs`. Same table plus same algorithm equals identical
output by construction. Every mapping is a strict subset of official NFKC.
Each is limited to mappings whose target is plain ASCII. Those are the
mappings that improve matching for an ASCII-English corpus.

`fold(_:)` rewrites a string scalar by scalar. It replaces each mapped
character and passes everything else through unchanged. `foldScalar(_:)`
holds the actual mapping. An explicit literal table comes first. Then come
arithmetic range families: full-width forms, super and subscripts, circled
letters, and Roman numerals. These are expressed as offsets, so the table
stays small and easy to check. Canonical recomposition, such as half-width
katakana, is deliberately out of scope. The file documents this so a
future reader does not mistake the omission for a bug.

## Normalizer.swift

This file provides `Normalizer`, the canonicalization front door. It runs
before stemming and lexicon lookup. This lets different surface spellings
of the same word collapse to one key.

`Normalizer.normalize(_:)` applies the `NFKCSubset` fold, then a
Unicode-aware lowercase. The fold runs first for a reason the code spells
out. A full-width `Ａ` must first become `A`. Only then does it fold to
`a`, matching the key that a plain `a` produces. Lowercasing alone would
leave the full-width form distinct. The same word would then split into
two different lexicon keys. The function is deterministic. It is
byte-identical to the Rust port for every input in the shared conformance
fixture.

## Stemmer.swift

This file provides `Stemmer`, a hand-ported implementation of the Porter2
(Snowball English) stemming algorithm. Stemming reduces a word to its
root: "dogs" becomes "dog," and "classification" becomes "classif." The
point is recall. Text about "running" and text about "runs" should reach
the same lexicon entry.

Porter2 removes English suffixes in five ordered steps. Two regions of the
word, called R1 and R2, guide this process. Vowel-consonant transitions
compute these regions. Most removals only apply inside a region. This
prevents overstripping short words. Two exception lists handle irregular
forms. For example, "skies" becomes "sky" rather than "ski."

`Stemmer.stem(_:)` runs the full algorithm. First comes preprocessing:
apostrophe marking and consonant-y marking. Then comes region computation,
then steps zero through five. Words shorter than three letters return
unchanged, per the specification. `Stemmer.bundledReferenceCorpus()`
exposes the pinned test corpus, `Resources/SnowballEnglish.json`. The
conformance test uses it to verify a claim. This port and the Rust port,
which uses the `rust-stemmers` crate, must produce byte-identical stems
for every input.

## WordClass.swift

This file provides `WordClass`. This is the label produced by tagging Step
1 of the encoder: `.noun`, `.verb`, or `.other`.

Step 1 keeps only nouns and verbs. They carry the subject matter of a
sentence. `.other` is the discard bucket for everything else: articles,
prepositions, adjectives, punctuation. The enum is backed by strings. So
it serializes to a stable, human-readable JSON form: `"noun"`, `"verb"`,
`"other"`. Both the shared conformance vectors and the Rust port read this
form.

## WordClassTable.swift

This file provides the noun and verb fast-path table. It also provides its
live-swappable process cache. The table is the first stop for every
token. A token present in it resolves to its word class in constant time.
No tagger is invoked in that case.

Three load functions cover the table's life cycle. `loadBundled()` reads
the pinned snapshot shipped in `Resources/WordClassTable.json`.
`loadWritable()` reads the merged artifact that `PoolReducer` maintains
outside the app bundle. `loadWithPrecedence()` is the canonical entry. It
prefers the writable merged artifact, because that one contains learned
tokens. It falls back to the bundled pristine table only when needed.

`WordClassTableCache` solves a harder problem. It adopts a new table while
the process keeps running. Waiting for a process restart would delay
learning indefinitely after the reducer merges new tokens. The cache
instead publishes an immutable snapshot behind a lock. This snapshot is a
`WordClassTableSnapshot`: the parsed table plus precomputed membership
sets. Readers copy the snapshot reference out. They test membership
outside the lock. A swap can therefore never produce a torn read. A reader
sees the whole old table or the whole new one, never a mix. `swap(_:)`
publishes a new snapshot and bumps a version counter.
`reloadFromPrecedence()` and `reload(fromArtifact:)` re-resolve the table
from disk and publish it. The version counter matters for determinism.
Tagging is deterministic given the input and the table version. The
counter makes that version observable.

## WordClassTagger.swift

This file provides the public tagging API, `LatticeLib.wordClass`, in its
four overloads. This is encoder Step 1 as callers see it.

Every overload shares the same fast path. The token is lowercased. Then it
is checked against the live table snapshot: the verb set first, then the
noun set. A token listed in both resolves to verb. Only a token absent
from the table, a novel token, reaches a tagger.

`wordClass(_:)` is the default path. Novel tokens go to the deterministic
HMM tagger on all platforms, including Apple. This rule keeps the Swift
and Rust ports bit-identical. Apple's NLTagger is a black box that can
change between OS releases. So it is never in the default path.

`wordClass(_:tagger:)` threads an explicit estate choice. An estate is one
user's memory store. Each estate picks its novel-token tagger once, at
creation. The `.hmm` choice is the cross-platform baseline. The
`.nlTagger` choice opts in to Apple's NaturalLanguage tagger. It buys
higher accuracy at the cost of cross-platform determinism.
`taggerEnabled(osVersion:minOSVersion:)` gates that choice. The table pins
the NLTagger OS version that seeded it. An older OS fails closed to
`.other` rather than invoke a differently behaving tagger.

`wordClass(_:recordNovel:)` and `wordClass(_:tagger:recordNovel:)` add
privacy control. By default, a novel token's tag is recorded into the
shared pool cache. This lets the table learn it later. Sometimes the text
being classified is private memory content. Callers then pass
`recordNovel: false`, and the side effect is suppressed. The returned tag
is identical either way. Only the recording changes. The shared cache
itself, `sharedNovelCache`, is wired to a real file writer only when the
`LATTICE_POOL_DIR` environment variable is set. Otherwise the submitter is
a no-op. No deployment leaks tokens to disk by default.

## HMMTagger.swift

This file provides the deterministic novel-token tagger. It is a
three-state Hidden Markov Model, over noun, verb, and other. Integer
Viterbi decodes it.

Why a statistical model here at all? Novel tokens are exactly the words no
table covers. So some generalization is required. Why integer arithmetic,
though? Floating-point rounding can differ across platforms and
compilers. This tagger must produce the same answer on every port. The
model weights are log-likelihoods. They are scaled by one thousand and
rounded to integers at build time. Runtime scoring is pure integer
addition and comparison.

The observation alphabet is morphological. Each token maps to one
observation, such as "ends in -ing" or "ends in -tion." Other options are
"contains a non-letter" or "plain." `observe(_:)` checks these in a fixed
priority order. Both ports replicate that order exactly. `tag(_:)` then
picks the state with the best initial plus emission weight. For a
one-token sequence, Viterbi reduces to that best-scoring pick. Ties
resolve to the lowest state index.

The weights ship in the frozen artifact `Resources/HMMTaggerModel.json`.
They are trained on the MASC 3.0.0 Penn Treebank annotation. The training
deliberately uses only the corpus's rare words, at frequency one. The
tagger only ever sees out-of-vocabulary words. Rare words are the best
statistical proxy for those words. This gives the correct noun-leaning
prior for unknown words. Estimating from the full corpus would let
frequent function words dominate instead. It would then wrongly default
unknowns to `.other`.

## NovelTokenTaggerChoice.swift

This file provides `NovelTokenTaggerChoice`. This is the two-case enum,
`.hmm` and `.nlTagger`, that selects the novel-token tagger for an estate.

The type mirrors an identically named enum in PersistenceKit. There, the
choice is stored on the estate configuration. The two packages define the
type independently, so that neither depends on the other. A consumer
bridges between them with a trivial switch. The file documents the
threading rule: pass the choice down as a parameter from the estate
configuration to the call sites. Never pass it through global state.
`.hmm` is the default and the federation-safe choice. `.nlTagger` is
Apple-only. It trades determinism for accuracy.

## NovelTokenCache.swift

This file provides the local accumulation cache for novel-token tags. It
also provides two wire-format types, `PoolEntry` and `PoolSubmission`.

When the tagger classifies a novel token, the result is worth keeping. If
many devices see the same unknown word, the shared table should learn it.
`NovelTokenCache.record(token:wordClass:)` appends the tagged token to a
pending list under a lock. At exactly `poolSubmitThreshold`, fifty
entries, the cache builds a `PoolSubmission`. It is stamped with the table
version, platform, and tagger version. The cache then drains the list. It
hands the payload to an injected submitter closure outside the lock.
Entries below the threshold are kept indefinitely. There is no aging or
cleanup, by contract.

The submitter is injected at construction. Tests can then assert the
drain without touching the file system. Hosts that never configure a pool
get a no-op this way. The version stamps exist for a reason. The pool
consumer discards submissions made against a stale table version. An
observation is only meaningful relative to the table that failed to
contain it.

## NovelPoolSubmitter.swift

This file provides the production submitter factory. It turns a drained
`PoolSubmission` into a JSON file in a local pool directory.

The pool is a directory of files, not a database or a network call. Files
are cheap, observable, and easy to reason about. A future reducer can
process them in name order. Each file name embeds an ISO 8601 timestamp
plus a short unique suffix. Writes are atomic and fire-and-forget. A
failed write is logged, and the data is discarded for that cycle. It is
never retried and never crashed on. Pool data is recoverable from future
observations.

`make(poolDirectory:)` returns a submitter closure bound to a directory.
`makeDefault()` binds to the resolved default. `poolDirectory()` resolves
that default. It checks the `LATTICE_POOL_DIR` environment variable first.
Otherwise it uses an Application Support path on Apple platforms, or an
XDG data path elsewhere. This function is public for a reason. The write
side is the submitter. The read side is the governor that triggers the
reducer. The two sides must agree on the path. So this function lives
here as the single source of truth. `tableArtifactURL()` resolves the
sibling location of the writable merged table. That table is
`WordClassTable.json`, next to the pool directory. This is the file the
reducer updates and `WordClassTable.loadWritable()` reads.

## PoolReducer.swift

This file provides the batch job that closes the learning loop. It reads
pooled submissions. It merges qualifying novel tokens into the writable
word-class table artifact.

`PoolReducer.reduce(poolDirectory:tableArtifactURL:now:maxFiles:)` runs in
steps. It first seeds the writable artifact from the bundled table if none
exists. The reducer cannot write into the read-only app bundle otherwise.
It then loads the table and enumerates `pool_*.json` files. It processes
the oldest `maxFiles` of them. Bounding the batch keeps a large backlog
from stalling the caller. The remainder drains on later runs. Each file is
decoded and version-checked. A malformed or stale-version file moves to a
quarantine directory rather than being deleted. This preserves forensic
value. Within the run, the first occurrence of each token wins. Tokens
already in the table are skipped. Only NOUN and VERB tags expand the
table, since an OTHER tag is already represented by absence. Consumed
files move to an archive directory. This makes the reducer idempotent, so
re-running on a drained pool is a no-op. If anything was consumed, the
artifact is rewritten atomically with sorted keys and an advanced
snapshot date.

`PoolReduceResult` summarizes a run: consumed, quarantined, added, and
skipped counts. `PoolReducerError` covers three failure classes: table
read, table write, and pool directory unreadable. The file warns against
wiring the reducer into the per-token tagging path. Reduction is a batch
operation. A resident governor triggers it on an idle cadence, or an
operator triggers it by command.

## Lexicon.swift

This file provides the canonicalization lexicon type and its deterministic
builder. The lexicon is encoder Step 2's reference data. It is a flat map
from `stem(normalize(token))` to a concept identity.

`CanonicalizationLexicon` is the pinned artifact shape: a version, a
language, and the entries map. The version is part of the agreement
protocol. Two encoders must share it, or their concept bags can diverge.

`LexiconBuilder.build(_:)` constructs the artifact from two public
sources. Wikidata supplies the concept identities. It maps surface forms
and aliases to Q-IDs, plus the property P8814, which links WordNet synsets
to Q-IDs. WordNet supplies word senses in frequency order. The build
considers every candidate key with a three-tier rule, and the lower tier
wins. First comes a Q-ID reached through a WordNet sense of the word. This
is where WordNet disambiguates: "dog" maps to the animal, not the
sausage, since the animal is the primary sense. Next comes a Q-ID known
only from a Wikidata alias. Last comes a `wn:` synset fallback, used only
when no Q-ID exists for the concept. Within each tier, deterministic
tie-breaks apply in order. The tie-breaks are sense rank, support count,
and lowest Q-number. These make the result order-independent. Same
inputs and the same artifact yield the same output on any machine, byte
for byte. Keys are
derived with the same normalize-then-stem pipeline the runtime uses. So
build-time and runtime keys agree exactly. Only single-token surfaces
become keys, because the runtime looks up one token at a time.

## ConceptBag.swift

This file provides encoder Steps 1 through 3. `BagBuilder` turns a block
of text into a concept bag, the weighted map from concept identity to
occurrence count.

`bag(_:lexicon:keep:)` makes one pass over the tokens. Each token is
normalized and stemmed, then looked up in the lexicon. The token is kept
if its word class is in the keep set, nouns and verbs by default. It is
also kept if it resolves to a Wikidata Q-ID. That second condition is a
deliberate relaxation. Part-of-speech taggers often mislabel named
entities. A proper noun like "Everest" matters more to classification
than most common nouns. Deciding membership from the pinned lexicon keeps
the relaxation deterministic. The lexicon is identical everywhere, while
cross-platform proper-noun tagging is not. A kept token contributes its
concept identity when the lexicon has one. Otherwise it contributes its
bare stemmed surface form. A surface form can still match a signature
carrying the same string.

`bag(_:lexicon:keep:recordNovel:)` is the privacy variant. It produces a
byte-identical bag. But when `recordNovel` is `false`, novel tokens found
during tagging are not accumulated into the shared pool cache. This is the
seam the memory-capture path uses. Private text never reaches the pool
pipeline this way. `bag(_:lexicon:keep:taggerChoice:)` is the
estate-configuration variant. It threads the estate's explicit tagger
choice into Step 1. The bag is then built consistently with the estate's
other indexed content.

## Code.swift

This file provides the FDC code grammar: what makes a code string
well-formed.

A code is a three-digit integer, from 000 through 999. It may carry an
optional decimal extension of one to eight digits, such as `540` or
`540.137`. Each extension digit subdivides its parent into ten. So the
extension carries leaf resolution under a spine code. The eight-digit cap
bounds the printable form at twelve characters. Tools can then reason
about column widths and string lengths.

`Code.isWellFormed(_:)` checks the grammar and nothing else. Validity is
purely grammatical by design. A well-formed code is accepted, stored, and
round-tripped even if no term currently encodes to it. Whether a code is
"known" belongs to the caller's own known-code set, not to the grammar.
`Code.integerBase(of:)` extracts the three-digit integer base of a
well-formed code. It returns `nil` for a malformed one.

## FDCFrame.swift

This file provides the frame model, the versioned list of codes and
labels, plus the ancestry math over code strings.

`FDCEntry` is one code with its heading label. The label is preserved
verbatim, including subject markers, because build tooling consumes the
raw heading text. `FDCFrame` is the versioned list. Ancestry is not stored
anywhere. It is derived from the code string itself instead.

The derivation has two regimes. The file's long comment explains why a
naive dot-split is wrong. The three-digit integer head is a Dewey-style
positional hierarchy. It reads at the hundreds, tens, and units places.
The parent of `006` is `000`. The parent of `510` is `500`. The decimal
tail, though, is a per-segment hierarchy. The parent of `006.6` is `006`.
`decimalParent(of:)` implements both regimes as a pure function of the
string. `ancestors(of:)` walks parents upward. It returns the chain
root-first, so `ancestors("006.6")` is `["000", "006"]`. `children(of:)`
filters the frame's codes by parent identity. It sorts the result for
deterministic output.

## CodeSignature.swift

This file provides the build-time signature assembly. It shows how a
code's characteristic term set is put together before it ships as an
artifact.

Every code in the frame has three source texts: its label, the title of
its reference article, and the article body. `SourceWeights` pins their
relative importance. The label gets a weight of three, the title two, the
article one. The label is the most precise statement of what a code
means. The article body is the broadest.
`SignatureAssembler.merge(label:title:article:weights:)` combines the
three concept bags into one weighted term map.

`SignatureAssembler.accumulateAncestors(ownTerms:ancestorsOf:)` then adds
every ancestor's own terms into each code's signature. A specific code,
such as `540.1`, thereby carries the full vocabulary of its lineage. So
text that talks about the general subject still supports the specific
code during descent. The result is a `CodeSignature` per code. The
`fingerprint` field stays `nil` until a later SimHash pass computes it,
once the global concept vocabulary is known. All of this is build-time
machinery. The runtime ships only the compact term sets.

## FDCMatcher.swift

This file provides encoder Steps 4 and 5. It scores a concept bag against
the signatures. It descends the frame to the most specific supported
code. This file is the heart of the classifier.

`FDCMatcher.init(lexicon:frame:signatures:stopThreshold:scoreMode:)` builds
the runtime index once. Every term across all signatures is interned. Each
is sorted alphabetically and assigned a dense integer identifier. The hot
path then works entirely with integer keys. Profiling showed this removes
the dominant cost, string hashing, on large imports. The identifier order
is deliberately the alphabetical order. So sorting integer sets visits
terms in the same sequence as sorting the original strings. Floating-point
sums are computed in the same order this way. They stay bit-identical to
the pre-interning implementation. The initializer also builds an inverted
index, from term to codes. It builds document frequencies for IDF
weighting and per-signature norms too.

`ScoreMode` selects how overlap becomes a score. `.raw` sums the bag
counts of shared terms. `.idf` weights each shared term by its rarity
across signatures, using inverse document frequency. So distinctive
vocabulary counts for more than vocabulary that appears everywhere.
`.cosine` and `.idfCosine` also divide by a signature-size norm. This
removes the advantage of big signatures. The shipped runtime uses `.idf`.
This measurably improved code selection over `.raw` on the v1.0 frame.

`encode(_:)` and the two `encodeAnchor` overloads are the public entries.
The `recordNovel: false` variant suppresses pool accumulation for private
text. All delegate to one private method, `encodeFromBag(_:)`, which runs
the algorithm. Step 4 collects candidate codes from the inverted index. It
scores each and takes the best. Ties break toward the lowest code, in a
fixed scan order, so the result never depends on hash ordering. Two
guards protect honesty here. An empty or non-matching bag returns `nil`,
meaning UNRESOLVED, never a guess. The other guard is
`maximumTiedWinnersForClassification`, set to four. When more than four
codes tie at the top score, the matcher also returns UNRESOLVED. A wide
tie means the text's
vocabulary is generic. Any single winner would be confidently wrong in
that case. Step 5 then walks down the frame. A child must carry at least
`stopThreshold` raw overlap to be a candidate. This cutoff is
mode-independent by design. The best-scoring child wins, and the walk
repeats until no child qualifies. The deepest reached code is the answer.

`dominantQID(_:)` computes the other half of the anchor. This is the
highest-counted Wikidata Q-ID in the bag. Ties break toward the lowest
Q-ID. The result is `nil` when the bag has none. This value answers "what
the text is most about." It surfaces in the same pass, so consumers fill
an anchor without re-encoding.

## FDCRuntime.swift

This file provides `FDC`, the runtime facade that consumers actually
call. It owns the pinned artifacts. It hides the matcher behind four
static functions.

There are three runtime artifacts: the lexicon, the frame, and the
compact signatures. They load once per process from the module bundle.
They load together, or not at all. The signatures loaded here are the
compact form, a map from code to term list, because the runtime matcher
uses only term membership. The weighted form
remains a build record. The loaded matcher is configured with
`scoreMode: .idf` and the pinned `stopThreshold` of one. Testing showed
this value is inert on the shallow v1.0 frame. It is pinned for the
contract rather than for tuning.

`FDC.encode(_:)` returns the lattice code, or `nil` for UNRESOLVED.
`FDC.encodeAnchor(_:)` adds the dominant concept Q-ID. The
`recordNovel: false` overload is the privacy seam for memory capture.
`FDC.isAvailable` reports whether the artifacts loaded. `FDC.dataVersion`
reports the signatures version, so callers can record provenance with
every classification. `FDC.ancestors(of:)` exposes the frame's ancestry
walk. Consumers get the chain this way, without reaching into `FDCFrame`
directly. `FDC.label(for:)` returns a human-readable heading for a code.
For three-digit codes it walks up one level first. Leaf labels at that
depth are often multi-subject compounds. The parent carries the cleaner
single-topic heading that dashboards want instead.

## QIDClosure.swift

This file provides the Q-ID ancestor surface. It takes a Wikidata Q-ID.
It returns every ancestor reachable through two relation types. These
are instance-of, property P31, and subclass-of, property P279.

Consumers use this to generalize a concept, from "dog" up through
"mammal" and "animal," without any network access. The edge graph is a
pinned, offline Wikidata snapshot, `Resources/QIDClosureEdges.json`. Build
tooling produces it, and it is checked in like the other artifacts. The
runtime never queries Wikidata.

`QIDClosure.ancestors(of:)` computes the transitive closure. An iterative
breadth-first walk over the edge map does the work. It excludes the
queried Q-ID itself. It returns the result sorted numerically by Q-number.
The walk dedupes visited nodes, so even a cycle in the source data would
terminate. Results are memoized per Q-ID for the life of the process. A
lock guards the memo, while the walk itself runs outside the lock. A race
between two threads computing the same Q-ID is harmless. Both produce the
identical pure result. `isAvailable` and `dataVersion` mirror the `FDC`
facade: artifact presence and provenance.

## LexRank.swift

This file provides the article-reduction step used at build time.
LexRank selects the most central sentences of a long article before the
encoder runs over it.

Code signatures are built from reference articles. But a whole article is
noisy. The useful vocabulary concentrates in its most representative
sentences. LexRank finds them with eigenvector centrality, the idea
behind PageRank, over a sentence-similarity graph. Sentences are nodes.
Two sentences connect when their stemmed term vectors are cosine-similar
above a threshold of 0.1. Sentences that many other sentences resemble
score high.

`LexRank.reduce(_:sentences:)` segments the text. It builds per-sentence
term-frequency vectors with the shared primitives. It assembles the
thresholded similarity graph. It then asks `SubstrateML.EigenvalueCentrality`
for scores. The function returns the top ten sentences by default, joined
in their original order. It returns the text unchanged when the text is
already short enough. This is build-time-only machinery. It never runs on
user devices, so it carries no cross-platform agreement constraint,
though it stays deterministic. It is also the one place the package uses
its single dependency, SubstrateML.

## Rust Port and Conformance

The `rust/` directory contains the second leg of the library. Seventeen
source files mirror the Swift implementations, from `tokenizer.rs`
through `fdc_runtime.rs`. The two legs share their pinned artifacts. They
are gated by shared conformance fixtures in `rust/tests/fixtures/`.
These are recorded input-output pairs for normalization, tagging, and
full FDC encoding. Both implementations must reproduce them byte for
byte. When you change either leg, run both test suites. The fixtures are
the contract.
