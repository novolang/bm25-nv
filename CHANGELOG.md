# Changelog

All notable changes to bm25-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-10-10

**The implementation.**  Every body the interface release published as
`todo()` is written, `novo test --isolate tests` is green over five
suites and 27 tests, no run reports a leak, and the merged per-line
coverage of `src/` is 340 of 341 instrumented lines — every line but
the one the marker below excuses.

### Added

- `bm25index` — the corpus counted once: the document lengths, the
  term frequencies, the document frequencies, the sorted vocabulary
  and the total.  `add` and `add_all` answer a NEW index and advance
  its generation by the number of documents added, so a table built
  before an add is refused rather than trusted.
- `bm25score` — the three saturations out of one shape: Okapi's, BM25+
  adding `delta` outside the saturation, and BM25L adding it inside,
  each written out in the module comment.  The IDF table carries the
  vocabulary, the values, the parameters it was built from and the
  generation it read.  Its three IDFs are the Lucene default, section
  3.2's Robertson formula with its negative branch intact, and
  `rank_bm25`'s floor over the values that came out positive.
- `bm25query` — the scores, the ranking, the cut, the precomputed
  table's two calls, the best hit and `unknown_terms`.  The ranking
  applies `bm25hit.compare`'s total order in a merge sort written out
  in the module.  The obvious spelling — `list.sort_by` with a key
  that builds `(a.score, a.doc)` — is the shape the compiled leg
  orders wrong, filed as
  `codegen-llvm/list-sort-by-with-a-pair-valued-key-orders-wrong-on-the-compiled-leg`
  and reproduced against 0.20.2 while this was written, so the
  hand-written sort is the workaround; it also fixes the tie order the
  standard sorts do not promise.
- `bm25params` — the defaults, the two per-variant defaults,
  `rank_bm25`'s own constructor defaults, the four `with_*` builders,
  the four-field `check`, and the name round-trips a configuration
  file needs.
- `bm25error` — each refusal is a value with a stable code and one
  sentence, built in one place so the free function and the `Error`
  trait method cannot drift apart.
- `tests/bm25_edge_tests.nv` — the refusals read back, every variant
  and IDF name, the reads that answer zero rather than refuse, the
  table's cut at a `k` of zero, and the deduplication of
  `unknown_terms`.
- `rank_bm25`'s own worked example reproduces exactly: under
  `bm25params.rank_bm25_defaults()`, the second document of the
  three-document corpus scores 0.93729472 and the other two score
  zero.
- The toolchain floor moves to 0.20.2, the compiler this release was
  built, tested and measured against.

### Changed

- **Three `== false` comparisons in the oracle suites are spelled
  `not`.**  The 0.20.2 linter raises `redundant-bool` on each and
  suggests exactly that rewrite, which is the same predicate.  All
  three are pre-existing — `tests/bm25index_tests.nv` and
  `tests/bm25score_tests.nv` carry the same one and two at HEAD — and
  the assertions' meanings are untouched.

### Known

- **A corpus of one or two documents leaves the floored IDF nothing
  positive to average.**  Every Robertson IDF over such a corpus is
  at or below zero, so `epsilon` times the mean of the positive ones
  is a mean of no values and the substituted value is zero.  Measured:
  over `[["a"]]` under `rank_bm25_defaults()`, `bm25score.idf` answers
  0.0 and `bm25query.score_all` answers `[0.0]`, so such a corpus
  ranks every query at zero.  A single document is degenerate for
  BM25 whatever the IDF does, and the doc comment on
  `Bm25IdfFloored` names the reading this follows.
- **One line under `src/` is excused from the coverage measurement**:
  the refusal arm of `bm25query`'s private `doc_score`.  The table it
  is handed is built from the index it walks, so neither of
  `bm25score.score_with`'s refusals can fire there, and the line
  carries a `// cov: skip` marker with that reason.
- **Three lint warnings stand in the interface suites.**  `novo.lint`
  reports `redundant-bool` on three `== false` comparisons —
  `tests/bm25index_tests.nv:67`, `tests/bm25score_tests.nv:217` and
  `tests/bm25score_tests.nv:229` — each of which the linter would
  rewrite as `not`.  They are the interface release's own, at the same
  sites in the commit this release starts from, and the suites are the
  published oracle, so they are left as they are rather than
  respelled.  Every module under `src/` and the new suite lint with no
  warnings.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `bm25index` — the corpus counted once as a value, and the decision
  the rest of the package follows from: an `add` answers a NEW index
  and it does not change only the new document. The corpus size
  grows, the average document length moves, and the document
  frequency of every added term goes up, so EVERY inverse document
  frequency changes and every score computed before the add is a
  different number from the one the same call answers after it. The
  reference implementation cannot meet this because its index cannot
  change; a package whose index can has to say something about it, and
  saying nothing means a cached scorer keeps ranking by the old
  numbers, plausibly and silently. So the index counts its
  generations, `generation` is published, and `bm25score.idf_table`
  records the generation it read. Nothing here is a comment in a
  document: `score_with` and `top_k_with` compare the two numbers and
  answer `Bm25StaleIdfTable` with both of them named.
  The corpus arrives as `[[Str]]` and there is no tokenizer, because
  the tokenizer decides recall and decides it invisibly: a package
  that shipped one would make two corpora built on two different
  defaults produce scores that look comparable and are not. The
  postings are stored FORWARD, by document, which is `rank_bm25`'s own
  shape and costs a whole-corpus pass per query; the inverted
  direction is named in the README as missing rather than left for a
  profile to find.
- `bm25params` — the variant is a FIELD of the parameters value, not
  an argument to the scorer. Three reasons, in increasing cost.
  `delta` is read by BM25+ and BM25L and means a different quantity in
  each, and is not read at all by plain Okapi, so a variant chosen at
  the call site leaves `delta` tuned for a formula this call is not
  using. The inverse document frequency is not independent of the
  variant: Lv and Zhai's lower bound is only a lower bound when the
  IDF is positive, so the two have to be chosen together and a value
  that held one and not the other would let them disagree. And a
  ranking is a comparison — with the variant as an argument, a loop
  that scored document 0 under Okapi and document 1 under BM25+ would
  compile, run, and sort. With it in the value the scorer reads, one
  ranking is one formula by construction, and the `Bm25IdfTable`
  carries the whole value so the precomputed path inherits the
  property.
  The IDF is published as a choice rather than defaulted in silence.
  Section 3.2 of Robertson and Zaragoza 2009 gives
  `log((N - n + 0.5) / (n + 0.5))`, which is NEGATIVE for a term in
  more than half the documents — under which a document containing
  that term scores below one that does not contain it at all, because
  the latter contributes exactly zero. On a small corpus half the
  vocabulary is above that threshold. `rank_bm25` papers over it with
  a floor whose substituted value is a property of the corpus and not
  of the term; that behaviour is here as `Bm25IdfFloored` because a
  port has to reproduce it, and `Bm25IdfRobertson` is here because an
  experiment reproducing published numbers needs the formula the
  numbers came from. The default is `Bm25IdfLucene`, which is positive
  everywhere and needs no tuning constant.
  `check` is a start-up call and nothing in the scoring path calls it:
  checking four floats per document per query is a cost paid a million
  times for a mistake made once.
- `bm25score` — the three variants are one shape with two knobs, and
  the module comment writes all three out so a reader can see that
  BM25+ adds `delta` outside the saturation and BM25L inside it. The
  IDF table is a value because a query of five terms over ten thousand
  documents is otherwise fifty thousand logarithms for a ranking that
  needs five; it carries the `Bm25Params` it was built from, which
  makes scoring with one formula's table and another's constants
  unrepresentable rather than merely discouraged.
- `bm25hit` — a hit is a document index and a score, never a copy of
  the document, because a ranking produces one per document and the
  caller wants forty. `compare` is TOTAL: score descending, then
  document index ascending. Ties are ordinary rather than exotic — a
  one-term query over equal-length documents ties every match, and an
  all-unknown query ties the whole corpus at zero — and a comparison
  that looked only at the score would leave their order to the sort,
  which drops and repeats rows between pages and opens the wrong file
  in a picker.
- `bm25query` — four shapes over one corpus, and `unknown_terms`,
  which is the one no reference implementation publishes. A query
  whose terms the corpus does not hold scores every document zero,
  `rank` then answers the whole corpus in document index order and
  `top_k` cuts it to the first `k`: sorted, complete, plausible, and
  carrying no information. The usual cause is a tokenizer
  disagreement, and no score can report it because zero is also the
  honest score of a document that does not match. So the check is a
  published call, it costs one document frequency lookup per query
  term, and the README names running it as a rule.
- `bm25error` — four refusals, with `code` stable across releases and
  `is_setup_fault` separating what a start-up check would catch from
  what depends on the call. Each is a value rather than a panic or a
  nan; a nan compares false against everything, so a nan score sinks
  to wherever the sort puts it and the ranking comes out subtly wrong
  with nothing in any log.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented:
  bm25-nv.<module>.<fn>`.
- **The package makes no device claim, and the scoring half could not
  honour one.** Every row is `[]` and the layer rules hold, but an
  inverse document frequency is a natural logarithm and `std.math.log`
  is refused at `@tier(embedded)`, along with every other
  floating-point routine in the standard library. `bm25index` is
  integer work and would build for a microcontroller; `bm25score` and
  `bm25query` would not. The README says so under "Running on a
  microcontroller" rather than leaving it to the link step.
- **A whole-corpus pass per query.** The postings are forward, by
  document, so `score_all` is proportional to the corpus and not to
  the query's postings. That is `rank_bm25`'s complexity and it is a
  deliberate match: an inverted index is a different value with a
  different build cost and a different incremental-add story, and
  adding one later is a `0.2.0` with a new type rather than a change
  to any signature here.
- **`bm25score.idf` recomputes the floor from the whole vocabulary**
  under `Bm25IdfFloored`, because the floor is a property of the
  corpus rather than of the term. A caller asking about more than one
  term takes `idf_table`, and the doc comment says so.
- **Toolchain defect found while staging, filed and not worked
  around.** `Result<T, E>` requires `E: Error` (SPEC § 3.4, E2018) and
  the compiler enforces it only when `E` is declared in the same
  module as the signature; a cross-module error type is accepted with
  no impl at all. This package's error type lives in its own module,
  which is the shape the publishing guide asks for, so the hole is
  not worked around here and no code changed because of it. Filed
  against the toolchain as
  `type-system-meta/result-error-bound-unenforced-across-modules`.
