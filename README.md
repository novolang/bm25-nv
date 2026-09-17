# bm25-nv

BM25 is a ranking function: given a query and a collection of
documents, it scores each document for how well it answers the query.
It was developed for the Okapi retrieval system and is specified by
Stephen Robertson and Hugo Zaragoza in
["The Probabilistic Relevance Framework: BM25 and Beyond"](https://www.staff.city.ac.uk/~sbrp622/papers/foundations_bm25_review.pdf),
Foundations and Trends in Information Retrieval 3(4), 2009. It is the
default ranking function of Lucene, Elasticsearch and most search
engines that are not learned rankers. This package brings it to
novo-lang. The reference implementations are the Python package
[`rank_bm25`](https://github.com/dorianbrown/rank_bm25) and
[Lucene's `BM25Similarity`](https://lucene.apache.org/core/9_0_0/core/org/apache/lucene/search/similarities/BM25Similarity.html).

**Status: NOT IMPLEMENTED — interface only.** Every function is
declared with its full signature, but every body is a `todo()` that
panics when called. The package is published so its design can be
reviewed and depended on before it is implemented. Version 0.1.0 will
be the first working release.

## What BM25 is

A **term** is one unit of text, already split out: a word, a stem, a
number. A **document** is a list of terms. A **corpus** is a list of
documents. A **query** is also a list of terms. This package works in
terms and never in strings of text; splitting text into terms is the
caller's job, and the section "What is not included" says why.

BM25 scores one document for one query by adding up, over the query's
terms, a weight for each term the document contains. Each term's
weight is a product of two numbers.

The first is the **inverse document frequency**, or IDF. It is large
for a term that appears in few documents and small for one that
appears in many, so a rare term is worth more evidence than a common
one. Robertson and Zaragoza give it in section 3.2 as
`log((N - n + 0.5) / (n + 0.5))`, where `N` is the number of documents
and `n` is the number of them containing the term.

The second is a **saturating, length-normalised term frequency**. Two
observations shape it. A term that occurs ten times in a document is
better evidence than one that occurs once, but not ten times better,
so the count saturates; the constant `k1` says how quickly. And a term
that occurs five times in a long document is weaker evidence than one
that occurs five times in a short one, so the count is divided by the
document's length relative to the average; the constant `b` says how
much. Section 3.2 gives the combination as
`tf * (k1 + 1) / (tf + k1 * (1 - b + b * dl / avgdl))`, where `tf` is
the number of occurrences, `dl` is the document's length in terms and
`avgdl` is the average document length in the corpus.

Two later papers correct one behaviour of that formula. As a document
grows, the length normalisation drives the weight of a term it
contains towards zero, so a long document that contains every query
term can score below a short one that contains fewer. **BM25+**, from
Yuanhua Lv and ChengXiang Zhai, ["Lower-Bounding Term Frequency
Normalization to Improve BM25"](https://sifaka.cs.uiuc.edu/~ylv2/pub/cikm11-lower-bound.pdf),
CIKM 2011, adds a constant `delta` to the normalised term frequency,
which puts a floor under that weight. **BM25L**, from the same authors
in ["When Documents Are Very Long, BM25
Fails!"](https://sifaka.cs.uiuc.edu/~ylv2/pub/sigir11-bm25l.pdf),
SIGIR 2011, adds the constant before the saturation instead of after
it.

| Constant | Default here | Meaning |
| --- | --- | --- |
| `k1` | 1.2 | How quickly a repeated term stops adding weight. At 0 a term counts once however often it occurs. |
| `b` | 0.75 | How much a document's length is divided out. At 0 length is ignored; at 1 the term frequency is fully divided by the relative length. |
| `delta` | 1.0 for BM25+, 0.5 for BM25L | The floor the two corrections put under a matched term's weight. Not read by plain BM25. |
| `epsilon` | 0.25 | `rank_bm25`'s floor multiplier. Read only under the floored IDF. |

Scores are floating point. A score has no upper bound and no unit, and
scores from two different corpora, two different parameter sets or two
different versions of the same corpus are not comparable with each
other. A score is only ever meaningful against other scores from the
same call.

## Install

```
novo pkg add bm25-nv
```

## Example

```novo
use bm25index
use bm25params
use bm25query

fn main() [io]
    // Three documents, each already split into terms by the caller.
    let corpus = [["hello", "there", "good", "man"],
                  ["it", "is", "quite", "windy", "in", "london"],
                  ["how", "is", "the", "weather", "today"]]
    // Count the corpus once. Every query below reads this value.
    match bm25index.build(corpus)
        Err(_) => println("the corpus cannot be indexed")
        Ok(ix) =>
            let query = ["windy", "london"]
            // Terms the corpus does not hold score nothing, so check
            // for them before reading the ranking.
            for term in bm25query.unknown_terms(ix, query)
                println("no document contains ${term}")
            // The two highest-scoring documents, best first.
            for h in bm25query.top_k(ix, bm25params.defaults(), query, 2)
                println("document ${h.doc} scored ${h.score}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: bm25-nv.<module>.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `bm25error` | The four refusals, a stable code for each, and which of them a start-up check would catch. |
| `bm25params` | The three BM25 variants, the three IDF formulae, the four constants, and a check over them. |
| `bm25index` | A tokenised corpus with the document lengths, the term frequencies and the document frequencies counted once. |
| `bm25score` | The formula: one term's weight, one document's score, and the table of inverse document frequencies a repeated query reads. |
| `bm25hit` | A ranked result: a document index, a score, and the total order over the two. |
| `bm25query` | Scoring a whole corpus against a query, ranking it, and taking the top of it. |

## How to choose an entry point

**`bm25query.top_k` is the ordinary call.** It scores every document
and answers the best `k`, which is what a search box does.

**`bm25query.best` answers one document.** Use it when the program
wants the single closest match and would otherwise index into a list.

**`bm25query.score_all` answers one score per document, unsorted.**
Use it when the scores are going into a column, a threshold test, or a
blend with another ranker.

**`bm25query.top_k_with` reads a precomputed table.** Build one with
`bm25score.idf_table` when many queries run against one corpus. It is
the only ranking call that answers a `Result`, because a table can be
older than the index.

**`bm25score.score` scores one document.** Use it when the document is
already chosen and the question is only how well it matches.

**`bm25score.term_score` is one term's contribution to one
document's score.** Use it to show a user why a document ranked where
it did.

## The rules a user needs

1. **The caller supplies the terms.** The corpus is `[[Str]]` and the
   query is `[Str]`. Terms are compared as exact strings, so `London`
   and `london` are two terms.
2. **The query must be tokenised the same way the corpus was.** A
   query term the corpus does not hold contributes nothing to any
   score. `bm25query.unknown_terms` is the call that reports them.
3. **A query of entirely unknown terms scores every document zero.**
   `bm25query.rank` then answers every document in document index
   order and `bm25query.top_k` answers the first `k` of them. That
   result is sorted, complete and carries no information.
4. **The variant is a field of the parameters, not an argument to the
   scorer.** One `Bm25Params` value is one formula, so every score in
   one ranking is computed the same way.
5. **The inverse document frequency is a choice with three settings.**
   `Bm25IdfLucene` is the default and is positive for every term.
   `Bm25IdfRobertson` is section 3.2's formula and is **negative** for
   a term that appears in more than half the documents; under it a
   document that contains such a term scores below a document that
   does not contain it at all. `Bm25IdfFloored` is `rank_bm25`'s
   behaviour: any IDF at or below zero is replaced by `epsilon` times
   the mean of the positive ones.
6. **`delta` means a different quantity in BM25+ and BM25L.** It is
   1.0 in the CIKM 2011 paper and 0.5 in the SIGIR 2011 one.
   `bm25params.defaults_for` hands out each variant's own value.
7. **`k1` must be at or above 0, and `b` between 0 and 1 inclusive.**
   `delta` and `epsilon` must be at or above 0.
   `bm25params.check` reports a value outside its range, naming the
   field. Nothing in the scoring path checks.
8. **An empty corpus is refused.** Every quantity BM25 is made of is
   undefined over one. A document of no terms is not refused: it is a
   legal document that scores zero and counts towards the average
   document length.
9. **`bm25index.add` answers a new index, and it changes every
   score.** The corpus size, the average document length and the
   document frequency of every added term all move, so every inverse
   document frequency moves with them. A score computed before the add
   is a different number from the one the same call answers after it,
   and the two cannot be compared or merged.
10. **A table built before an `add` is refused rather than used.**
    `bm25score.idf_table` records the index generation it read;
    `bm25score.score_with` and `bm25query.top_k_with` answer
    `Bm25StaleIdfTable` when it is not the index's.
    `bm25score.table_is_current` asks the same question first.
11. **A ranking is a total order: score descending, then document
    index ascending.** Two runs over the same corpus produce the same
    list, including where the ties fall and where `top_k` cuts.
12. **A document index is the document's position in the corpus handed
    to `bm25index.build`.** `bm25index.add` appends, so an index
    stays valid as documents arrive.
13. **Scores are compared exactly.** There is no tolerance and no
    rounding. A caller that wants scores within a tolerance treated as
    equal groups the list itself.

## Running on a microcontroller

This package makes no device claim and ships no device probe.

Every function in it performs no input or output, so the layer rules
hold. The counting half is another matter from the scoring half.
`bm25index` is integer work and would build for a microcontroller.
`bm25score`, `bm25query` and the `Float` fields of `bm25params` would
not: an inverse document frequency is a natural logarithm,
`std.math.log` is refused at `@tier(embedded)`, and so is every other
floating-point routine in the standard library. A program that wants
to rank on a device counts the corpus here and computes the scores
with fixed-point arithmetic of its own.

## What is not included

- **A tokenizer.** The corpus and the query are lists of terms. Case
  folding, stemming, stop-word removal and how punctuation is treated
  each change which documents a query can find at all, and none of
  those changes shows up in a score. Two corpora built with different
  settings produce scores that look comparable and are not, so the
  choice stays with the caller.
- **An inverted index.** The postings here are stored by document, so
  a query touches every document in the corpus however short the query
  is. A term-to-document posting list is what makes a ranking cost the
  length of the query's postings instead, and it is not here.
- **BM25F.** The fielded variant, which weights a term differently in
  a title than in a body, is section 3.4 of the framework paper. It
  needs a document to be several lists of terms rather than one.
- **Query term weights.** A query term that occurs twice counts twice.
  The framework paper's query term frequency saturation, section 3.2's
  `k3`, is not here, because a query long enough for it to matter is
  rare outside a relevance feedback loop.
- **Relevance feedback.** The framework paper's sections 3.3 and 4
  adjust the term weights from documents a user marked relevant. That
  needs a store of judgements, which is a different package.
- **Stored documents.** An index holds counts, not text. A hit is a
  document index and a score, and the caller looks the document up in
  whatever holds it.
- **Persistence.** An index is a value in memory. Writing one to a
  file costs `[fs]`, and this package declares no effects.

## Related packages

- [tokenizers-nv](https://novo-lang.org/packages/tokenizers-nv) splits
  text into terms. It is the step before this package, and the one
  this package deliberately does not do.
- [ahocorasick-nv](https://novo-lang.org/packages/ahocorasick-nv)
  finds where a set of literal strings occurs in a text. Take it to
  locate terms, this one to score documents that already hold them.
- [fuzzy-nv](https://novo-lang.org/packages/fuzzy-nv) scores an
  approximate match of one string against another for a picker. Take
  it when the input is a partial name and the candidates are short
  strings; take this one when the input is a query and the candidates
  are documents.
- [hnsw-nv](https://novo-lang.org/packages/hnsw-nv) and
  [vectorstore-nv](https://novo-lang.org/packages/vectorstore-nv) rank
  by embedding distance rather than by term overlap. A hybrid search
  runs both and blends the two scores; `bm25query.score_all` answers
  the column to blend.
- [spellcheck-nv](https://novo-lang.org/packages/spellcheck-nv)
  corrects a query term that no document contains.
  `bm25query.unknown_terms` answers the list to correct.

## Test vectors

```bash
novo test tests/bm25index_tests.nv   # the counts, and what an add moves
novo test tests/bm25score_tests.nv   # the papers' properties, and the stale table
novo test tests/bm25query_tests.nv   # rank_bm25's worked example, and the total order
novo test tests/bm25cover_tests.nv   # the parameters, the check, and the hit order
```

The normative worked example is the one on `rank_bm25`'s front page:
the documents "Hello there good man!", "It is quite windy in London"
and "How is the weather today?", each split on spaces, and the query
"windy London" split the same way. `BM25Okapi` scores them
`[0.0, 0.93729472, 0.0]` and returns the second document as the top
result. The suite asserts that vector under
`bm25params.rank_bm25_defaults()`, which is `rank_bm25`'s own
constructor defaults rather than this package's.

The other assertions are the properties the specifications state, not
transcribed decimals. From section 3.2 of the framework paper: at
`b = 0` a long and a short document with the same term count score
alike, and at `b = 0.75` the longer scores less; at `k1 = 0` one
occurrence and three score alike, and at `k1 = 1.2` three score more
than one but less than three times as much; and the Robertson IDF is
negative for a term in three documents of four, so a document
containing that term scores below one that does not. From the CIKM
2011 paper: BM25+ scores a matched term at least `delta` times its
IDF, whatever the document's length. From the SIGIR 2011 paper: BM25L
lifts the same term less than BM25+ does, because its shift saturates.

The suite also asserts that an empty corpus is refused, that an empty
document is not, that `add` moves the average document length and the
document frequencies, that a table built before an `add` is refused
with both generations named, that equal scores come out in document
index order, and that a parameter outside its range is reported by
`bm25params.check` rather than by a ranking that runs.

The tests compile today and fail at run, each on the
`not implemented: bm25-nv.<module>.<fn>` panic that is its body. That
is the expected state of an interface release. They turn green one at
a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `bm25error.Bm25Error` and the other public types | the types are declared |
| `bm25error.message`, `.code`, `.is_setup_fault` | no |
| `bm25params.defaults`, `.defaults_for`, `.rank_bm25_defaults` | no |
| `bm25params.with_variant`, `.with_idf`, `.with_k1`, `.with_b` | no |
| `bm25params.with_delta`, `.with_epsilon`, `.check` | no |
| `bm25params.variant_name`, `.variant_named`, `.idf_name`, `.idf_named` | no |
| `bm25index.build`, `.add`, `.add_all` | no |
| `bm25index.doc_count`, `.doc_len`, `.avg_doc_len`, `.total_terms` | no |
| `bm25index.term_freq`, `.doc_freq`, `.contains_term`, `.vocabulary` | no |
| `bm25index.generation` | no |
| `bm25score.idf_table`, `.table_generation`, `.table_is_current` | no |
| `bm25score.idf`, `.term_score`, `.score`, `.score_with` | no |
| `bm25hit.hit`, `.compare`, `.rank_of` | no |
| `bm25query.score_all`, `.rank`, `.top_k`, `.top_k_with` | no |
| `bm25query.best`, `.unknown_terms` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
