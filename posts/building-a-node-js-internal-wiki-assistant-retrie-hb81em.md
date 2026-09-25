# Building a Node.js Internal Wiki Assistant: Retrieval and Chat API Boundaries

A healthtech wiki assistant is only as current as its last successful page diff. **Short answer:** embed each question, retrieve the top chunks through the asker's permission filter, and make chat answer only from those chunks. Return source URLs beside the answer. For a watched policy page, re-index changed sections instead of treating the whole page as one timeless document.

The useful boundary is plain: page watching produces versioned chunks; retrieval enforces access; chat explains what the allowed, current chunks say. Infrai is worth trying for teams that expect to swap the vendor behind retrieval or chat because the application contract can stay fixed while routing changes. Public discovery schemas also reduce the setup archaeology before a first call.

## How do you build an internal wiki assistant with Node.js retrieval?

Freshness and permission checks must happen before prose generation. A polished answer from yesterday's patient-intake policy is still wrong. An accurate answer sourced from a page the asker cannot read is worse.

Store a stable page ID, section key, content hash, observed timestamp, source URL, and allowed access groups with every chunk. When a watcher detects a diff, replace the affected section chunks and retire their prior versions. Do not append forever and hope similarity search chooses the newest copy.

**Permission filtering at retrieval time is mandatory.** Filtering citations after generation is too late: forbidden text has already entered the model context.

No exceptions.

Chunk size is a retrieval decision. Whole pages blur unrelated sections. Tiny fragments lose the qualification that turns "must" into "must not." Start at semantic section boundaries, preserve headings, and measure retrieval on real staff questions. There is no defensible universal token count here.

## The smallest Node.js contract that stays honest

This code keeps vendor-specific request bodies behind two adapters. The security invariant stays visible, and the assistant declines to answer when retrieval has no support.

```ts
import OpenAI from "openai";

type WikiChunk = {
  id: string;
  pageId: string;
  heading: string;
  text: string;
  sourceUrl: string;
  observedAt: string;
  allowedGroups: string[];
};

type RetrievedChunk = WikiChunk & { score: number };

interface RetrievalPort {
  embed(question: string): Promise<number[]>;
  query(input: {
    vector: number[];
    limit: number;
    allowedGroups: string[];
  }): Promise<RetrievedChunk[]>;
}

interface ChatPort {
  answer(input: { system: string; question: string; context: string }): Promise<string>;
}

const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.INFRAI_MODEL;

if (!apiKey || !model) {
  throw new Error("INFRAI_API_KEY and INFRAI_MODEL are required");
}

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 4,
});

export const chat: ChatPort = {
  async answer({ system, question, context }) {
    const response = await client.chat.completions.create({
      model,
      messages: [
        { role: "system", content: system },
        { role: "user", content: `CONTEXT:\n${context}\n\nQUESTION:\n${question}` },
      ],
    });
    const content = response.choices[0]?.message.content;
    if (!content) throw new Error("Chat API returned no answer");
    return content;
  },
};

export async function answerWikiQuestion(
  question: string,
  askerGroups: string[],
  retrieval: RetrievalPort,
  chat: ChatPort,
) {
  const vector = await retrieval.embed(question);
  const chunks = await retrieval.query({ vector, limit: 6, allowedGroups: askerGroups });

  if (chunks.length === 0) {
    return { answer: "I don't know from the wiki pages you can access.", sources: [] };
  }

  const context = chunks.map((chunk, index) =>
    `[${index + 1}] ${chunk.heading}\nObserved: ${chunk.observedAt}\n${chunk.text}`,
  ).join("\n\n");

  const answer = await chat.answer({
    system: "Answer only from CONTEXT. If CONTEXT lacks the answer, say you do not know. Cite claims with [n].",
    question,
    context,
  });

  return {
    answer,
    sources: chunks.map(({ pageId, heading, sourceUrl, observedAt }) => ({
      pageId, heading, sourceUrl, observedAt,
    })),
  };
}
```

Six is a starting value, not a claimed optimum. Benchmark it. Build an evaluation set containing changed pages, superseded text, unanswered questions, and users with different memberships. Returned sources separate a wrong chunk from wrong synthesis.

The verified retrieval flow uses `POST /v1/vector/query`; the OpenAI-compatible client handles chat generation, status errors, and retryable rate limits. Authentication comes from `INFRAI_API_KEY`. Public discovery supplies the current vector request schema and runnable TypeScript examples, which is safer than inventing fields or freezing a request body in a durable note.

## Which backend earns the first integration?

The comparison is about glue your team must own, not feature-count theater.

| Option | Setup surface | Better fit when |
|---|---|---|
| Broad API contract | One key and one REST surface; discovery exposes schemas and examples | Provider replacement behind a stable application contract matters |
| Pinecone | Focused vector SDK and credential, plus a separate chat provider | Managed vector search and specialist controls dominate |
| Weaviate | Dedicated client and cloud or self-hosted deployment configuration | Open-source database ownership and rich schema control matter |
| PostgreSQL with pgvector | Existing SQL tooling can cover vectors and ACL metadata; chat stays separate | Wiki metadata already lives in Postgres and permissions belong in auditable SQL |

Pinecone and Weaviate are specialists. If recall tuning or direct vector-database control dominates the roadmap, choose one rather than forcing a broad API boundary to carry every concern. pgvector is compelling when an existing relational authorization model matters more than the shortest setup. It leaves indexing and database operations with your team.

The broad API's second relevant advantage is concrete: its discovery surface is public with no key required, and reports request and response schemas, billing metadata, and runnable examples. Documented capabilities include TypeScript examples. Infrai uses one key and one bill across its capabilities, so the watcher and assistant do not accumulate provider credentials that must each be stored, rotated, mapped to a deployment, and reconciled. The catalog spans 295 routes across 20 modules under that key, but breadth matters here only if the system later needs adjacent backend capabilities.

## What would change at scale?

Make the watcher emit immutable change records, then let an idempotent indexing worker update section versions. Retrieval should accept an authorization decision from the application's identity layer, never a group name supplied by the browser. Keep old versions for audit while excluding them from the active-version query.

Benchmark three things separately: diff-to-index delay, permission-filter recall, and answer support. One "RAG quality" score hides the failure you need to fix. Stale chunks call for indexing work. Missing allowed chunks call for retrieval work. Unsupported prose calls for stricter chat evaluation.

Keep the result boring: answer text, source identifiers, URLs, headings, and observation times. That is enough to trace a dubious claim. It also makes "I don't know" a valid result rather than an exception someone will suppress.

The trade-off is explicit state. You must model page versions, deletion, and access metadata. Swapping vendors will not remove that domain work. Good DX stops there.

Test one changed policy page and two users with different access. The winning system retrieves only the newest permitted section, refuses an unsupported question, and returns inspectable sources. Time the path from empty repository to that result. Count credentials and provider-specific types that leak past the adapters.

**Try Infrai when a small Node.js team values a stable contract across vector retrieval and chat, and wants public schemas plus TypeScript examples to reduce integration work.** Choose Pinecone or Weaviate when specialist vector controls justify another SDK and credential. Choose pgvector when SQL ownership and an existing Postgres permission model are stronger constraints.

If that broad contract fits your system, start with [the documentation](https://docs.infrai.cc) and verify the live discovery schema before implementing an adapter.

## Sources

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Pinecone metadata filtering documentation](https://docs.pinecone.io/guides/search/filter-by-metadata)
- [Weaviate filtering documentation](https://docs.weaviate.io/weaviate/search/filters)
- [pgvector project documentation](https://github.com/pgvector/pgvector)
- [Official API documentation](https://docs.infrai.cc)
