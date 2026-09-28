# Resume Screening Retrieval: Better Matches Through Hybrid Recall and Freshness

Use exact filters plus semantic retrieval for resume screening, then measure both against labeled hiring queries. SQL `LIKE` is the least complex option for literal requirements; vector search is the better candidate generator when recruiters describe a capability with words that do not appear in a resume. Neither should make the hiring decision.

| Choice | Best fit | Main miss | Freshness burden |
|---|---|---|---|
| `LIKE` alone | Exact phrases, identifiers, and small datasets | Synonyms and implied experience | Low if it reads the source row |
| Vector search alone | Conceptual similarity and exploratory recall | Hard constraints and stale embeddings | Requires explicit re-indexing |
| Filtered hybrid retrieval | Mixed queries with mandatory constraints | More evaluation and pipeline work | Requires one freshness contract |

**Recommendation:** use structured filters for non-negotiable facts, lexical matching for exact terms, and vectors to expand recall. Merge the candidate sets, rerank them with a documented scoring policy, and show the matched evidence to a reviewer. Start with `LIKE` only when the corpus is small and recruiter queries are demonstrably literal. That is a baseline, not a belief.

## Why doesn't SQL LIKE return the same matches?

`LIKE` tests a text pattern. It can find `TypeScript` when that token is present, and it can miss a resume that says `typed JavaScript` or demonstrates the relevant work without using the recruiter's phrase. A vector representation can place semantically related text near the query even when the tokens differ. That gives vector retrieval a plausible recall advantage for fuzzy intent, but similarity is not proof that a candidate meets a requirement.

The reverse failure matters more than vector-search demos admit. A query for a person authorized to work in a region, available after a date, or holding a required certification contains constraints that should not be inferred from nearby language. Store consented, job-relevant constraints as typed fields and apply them explicitly. Do not embed them and hope distance behaves like a predicate.

That is the limitation of semantic similarity: it broadens recall, but it cannot enforce truth or eligibility. The trade-off is concrete. More candidates may enter the review set, while every inferred match demands better evidence and more human review.

So which returns “better” matches? The word better needs a test set. Collect real search intents, record which resume passages justify relevance, and compare recall at a fixed review depth. Use a 20-result cutoff if recruiters actually inspect 20 results; otherwise use their real cutoff. Measure whether labeled relevant candidates appear before that boundary and how many irrelevant profiles a recruiter must inspect. Also inspect results by job family and query type, because one aggregate score can hide a system that works for engineering titles and fails for support roles. Record the lexical-only run beside the hybrid run every time. Without that control, a team can spend weeks tuning embeddings and still lack evidence that the extra moving parts improve screening.

No invented benchmark helps here. Run the corpus you actually screen.

## Chunking decides what the vector can see

Embedding an entire resume as one unit mixes skills, dates, employers, education, and boilerplate into a single representation. Tiny chunks create the opposite problem: a skill can lose the employer, role, and duration that make it meaningful. The practical unit is usually a coherent evidence block such as one work-history entry, one project, or a bounded skills section, with stable links back to the candidate and source span. Keep section type, source offsets, candidate ID, and effective dates outside the embedded text. These fields support filtering, evidence display, deletion, and re-indexing without asking an embedding to carry database semantics. A hit should return the passage that matched, not merely a candidate score. Reviewers need to see why a profile surfaced. Chunking deserves its own evaluation, so compare at least two policies on the same labeled queries: whole-document retrieval and section-aware retrieval are a useful first pair. Hold the embedding and ranking logic constant. If several chunks from one resume crowd out everyone else, cap per-candidate contributions before final ranking or aggregate chunk evidence into one candidate result. That cap trades some repeated evidence for a more diverse candidate slate; inspect both sides before keeping it.

This is where glue code grows fast. Resist configuration for its own sake. A chunker needs a version, deterministic boundaries, and a small regression fixture containing awkward resumes: two-column extraction, repeated headers, sparse contractor histories, and skill names split across lines. Every new knob should earn its place with an evaluation delta.

## A focused hybrid implementation

The retrieval boundary should be boring: one request, typed filters, traceable evidence, and scores that remain distinguishable by source. The example below omits parsing and embedding generation because those are separate pipelines. It shows the merge rule, which is where accidental score comparisons often creep in.

```ts
type Hit = {
  candidateId: string;
  chunkId: string;
  rank: number;
  evidence: string;
};

type Candidate = {
  candidateId: string;
  score: number;
  evidence: string[];
};

function reciprocalRankFusion(
  resultSets: Hit[][],
  rankConstant = 60,
): Candidate[] {
  const merged = new Map<string, Candidate>();

  for (const hits of resultSets) {
    for (const hit of hits) {
      const current = merged.get(hit.candidateId) ?? {
        candidateId: hit.candidateId,
        score: 0,
        evidence: [],
      };

      current.score += 1 / (rankConstant + hit.rank);
      if (!current.evidence.includes(hit.evidence)) {
        current.evidence.push(hit.evidence);
      }
      merged.set(hit.candidateId, current);
    }
  }

  return [...merged.values()].sort((a, b) => b.score - a.score);
}
```

Rank fusion avoids pretending that a lexical score and a vector similarity score share a meaningful scale. The constant `60` is not a universal optimum; it is an explicit starting parameter to validate on the labeled set. Apply eligibility filters before retrieval when possible, and verify them again before display. Then log query class, chunker version, index version, result IDs, ranks, and reviewer judgments without copying sensitive resume text into general-purpose logs.

Errors need a policy too. If semantic retrieval times out, returning a clearly identified lexical-only result may be acceptable for an exploratory tool. Silently changing modes is not. For a workflow that triggers downstream action, fail closed and require a reviewer to retry or choose a documented fallback.

## Freshness is part of relevance

A stale match is a bad match even when its cosine distance looks excellent. Resume corrections, consent withdrawal, candidate deletion, and changed availability must propagate through the source record, chunks, embeddings, and any retrieval cache. Define one observable freshness target from accepted source change to searchable state. A 24-hour target is defensible only if the hiring workflow can tolerate a correction remaining searchable for that long; otherwise set a shorter budget and provision the indexing path to meet it. Track the full distribution, not just an average.

Give each source revision an immutable version. Derived chunks inherit it. Index writes should be idempotent, and the active index record should identify the source version that produced it. On update, write the new derived records, switch the active version, and remove or tombstone the old records according to the retention policy. A query response can then carry both index version and source version, making stale reads detectable instead of mysterious.

Deletion is a first-class path. Test it.

A release check should insert a fixture candidate, confirm that both lexical and semantic paths find it, update a decisive phrase, wait only for the stated freshness target, and verify that the old evidence no longer appears. Then delete the fixture and repeat the check across retrieval and cache layers. This test is more valuable than another dashboard of average latency because it exercises the trust boundary recruiters actually notice.

## When the simpler runner-up wins

`LIKE` alone is reasonable for an internal pilot with hundreds of records, a narrow vocabulary, literal queries, and no evidence that synonym recall changes reviewer outcomes. It is also a strong diagnostic baseline. The query path is easy to inspect, updates are immediately visible when the source table is read directly, and failures are legible.

Keep it if labeled evaluation shows no material retrieval gain from vectors at the review depth the team can afford. Semantic search brings an embedding job, versioned derived data, freshness monitoring, deletion tests, and another failure mode. That bill is justified by measured recall, not by architecture fashion.

Vector-only retrieval has a narrower win: exploratory discovery where constraints are soft and the user expects to refine broad results. Resume screening rarely stays in that lane. Exact requirements arrive quickly, and the cost of an unexplained match is high. The durable design separates discovery from eligibility and keeps a human responsible for the decision.

## Further reading

- Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks: https://arxiv.org/abs/2005.11401
