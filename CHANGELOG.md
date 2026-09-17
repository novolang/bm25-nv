# Changelog

All notable changes to bm25-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
