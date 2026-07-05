---
doc: OVERVIEW
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

# LatticeLib Overview

## What This Library Does

LatticeLib reads a piece of text. It answers one question: what subject is
this about? The answer is a lattice code. A lattice code is a structured
classification code. It gives a memory a place in an organized hierarchy of
subjects. Take the text "the dog chased a ball in the park." It encodes to
a code in the animals region of the hierarchy.

MOOTx01 is an on-device AI memory system. It stores what an AI observes over
time. It helps the AI recall that memory later. Every memory that enters
the system passes through LatticeLib first. LatticeLib assigns each memory
a code. That code becomes part of the memory's filing address. Memories
about the same subject cluster together this way. This holds even when the
memories share no exact words.

The engine inside LatticeLib is called FDC. FDC stands for Frame-Directed
Classification. A frame is a fixed, versioned list of classification codes
and their subject labels. FDC directs each piece of text to its best
position in that frame.

## The Problem It Solves

Two devices must file the same text under the same code. MOOTx01 estates
can federate. Federation lets separate devices share and compare memories.
Say one device filed "dog" under animals. Say another device filed it
under sports. Shared recall would then fall apart. Classification must be
deterministic for this reason. The same input must always produce the same
output. This must hold on every platform, every time.

Cloud classifiers cannot make this promise. They change without notice.
They need a network connection. They see private text. LatticeLib avoids
all three problems. It ships every piece of reference data it needs as
pinned artifacts. A pinned artifact is a data file built once and given a
version. It is checked into the repository and never changed at runtime.
The library runs entirely on the device. The same text against the same
artifact versions always yields the same code. This is the library's
agreement property.

The library keeps that promise across two independent implementations. A
Swift leg serves Apple platforms. A Rust leg, in `rust/`, serves everything
else. Shared conformance fixtures gate every release. These fixtures are
recorded input and output pairs. Both legs must reproduce them exactly.

## How It Works

Classification runs in five steps. The first three steps turn text into a
concept bag. A concept bag is a table. It maps each concept found in the
text to the number of times it appears. The last two steps match that bag
against the frame.

Step 1 tags each word and keeps only nouns and verbs. Other words carry
little subject meaning. Articles and adjectives fall into this group, so
they are dropped. A word the tagger has never seen is called a novel
token. A small, deterministic Hidden Markov Model tags novel tokens. This
step therefore never guesses differently on different platforms.

Step 2 canonicalizes each kept word. First the word is normalized. Odd
Unicode forms fold to plain letters in this step. Next the word is
stemmed, so "dogs" becomes "dog." The stemmed word is then looked up in
the canonicalization lexicon. The lexicon is a pinned artifact. It maps
about seventy thousand word roots to concept identities. A concept
identity is usually a Wikidata Q-ID. This is a public, language-neutral
identifier, such as `Q144` for "dog." Synonyms collapse onto one identity.
This is what lets two devices agree.

Step 3 accumulates the counts. The result is the concept bag.

Step 4 scores the bag against code signatures. A code signature is the set
of concepts that marks one classification code. It is prepared ahead of
time from reference articles. Some codes share concepts with the bag.
Those codes earn scores. Rarer shared concepts earn more. The best-scoring
code wins. Suppose nothing overlaps. Suppose too many codes tie instead.
The library then returns UNRESOLVED rather than guess.

Step 5 descends the frame. Starting from the winning code, the matcher
walks down to child codes. It keeps walking while the text still supports
the extra detail. It returns the deepest code the text still supports.

## How the Pieces Fit

Figure 1 shows the library's topology. It shows the major parts and how
data moves between them.

![Figure 1. Topology of LatticeLib](topology.svg)

*Figure 1. Topology of LatticeLib. Text flows left to right. It moves
through the shared text primitives into the concept bag. From there it
moves through the matcher to a code. Dashed regions mark the pinned
artifacts and the offline novel-token learning loop.*

The runtime entry point is the `FDC` enum. Consumers call
`FDC.encodeAnchor(text)`. This returns two values: the lattice code and the
dominant concept Q-ID of the text. The three runtime artifacts are the
lexicon, the frame, and the signatures. Each loads once per process.

Shared text primitives serve both the runtime encoder and the build-time
tools. These primitives are `Tokenizer`, `Normalizer`, `Stemmer`, and the
word-class tagger. Build-time tools produce the pinned artifacts. These
tools are `LexiconBuilder`, `SignatureAssembler`, and `LexRank`. They never
run on user devices.

One slow feedback loop improves the tagger over time. Novel tokens
accumulate in a local cache. When fifty gather, the cache writes them to a
local pool directory. A reducer later merges qualifying tokens into a
writable copy of the word-class table. The running process then adopts
the new table with an atomic swap. Classification stays deterministic
because determinism is always relative to a table version. The version
advances only at that swap.

Privacy is built into the seams. Sometimes the text being classified is
private memory content. Callers then pass `recordNovel: false`. No token
of that text is ever written to the pool. The pool itself is off by
default. It activates only when a host sets the `LATTICE_POOL_DIR`
environment variable.

## What Ships in the Package

The package ships three things: the Swift sources, the Rust port in
`rust/`, and the pinned artifacts. The artifacts live in
`Sources/LatticeLib/Resources/`. They are the frame (1,075 codes), the
code signatures (1,071 entries), and the lexicon (about 70,000 entries).
They also include the Q-ID ancestor graph (about 39,000 nodes), the
trained tagger model, the word-class table, and the stemmer conformance
corpus. Each artifact carries its own version string. The library reports
the versions it used. Every classification is reproducible this way, with
its provenance included.
