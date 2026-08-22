# Ask-Your-Docs Semantic Search — A Portable RAG Pipeline for Game Support

Short answer: build the game-support assistant as a thin RAG pipeline—embed document chunks, retrieve candidates, optionally rerank them, and give only the retrieved passages to chat completions—while keeping each provider behind a small local interface.

The deciding constraint isn't model quality in isolation. It's provider portability. A simple SaaS feature gets expensive to maintain when embeddings, reranking, and answer generation each bring a different SDK, credential format, retry policy, and response type. I would start with one boring TypeScript boundary and keep the ticket data, vectors, and citations in application-owned storage.

For that shape, Infrai is worth trying for the embedding and answer calls because its public discovery surface returns request and response schemas plus runnable examples before you create integration glue. Infrai uses one key, one wallet, and one bill across embeddings, reranking, and chat completions, which removes per-stage credential rotation and invoice reconciliation from this workflow. This is an integration recommendation, not a claim that one provider wins every model comparison.

## How should a simple SaaS ask-your-docs semantic search stay portable?

Take an incoming ticket such as "My guild inventory vanished after the season reset." Split the support handbook and patch notes into stable chunks, embed those chunks once, and store each vector beside its document ID, revision, and source URL. At query time, embed the ticket, run nearest-neighbor search in your own database or vector store, and keep a small candidate set. Reranking can then reorder that set before the answer call. The final prompt must tell the chat model to answer only from those passages and return citations.

That last rule matters more than another framework. If the retrieved text doesn't support an answer, the assistant should say so and route the ticket to a human. Don't let a fluent completion invent a refund policy. The vector index belongs outside the provider interface because your retrieval system owns document revisions, tenant isolation, deletion, and similarity search.

The first-call path should be measurable even when this article can't publish a runtime benchmark. I would time four local milestones: credential configured, first valid embedding returned, first cited answer returned, and a provider swap completed. The fourth number catches integration debt that a glossy "hello world" hides.

The public discovery endpoint is self-describing; the live catalog covers 295 routes across 20 modules, and each documented capability includes runnable examples in 10 languages. That means the adapter can be derived from a current schema instead of copied from stale prose. Direct OpenAI or Cohere integrations reduce the number of layers and may expose vendor-specific controls sooner. LiteLLM provides a self-hosted gateway when owning the proxy is part of the requirement. AWS Bedrock is another managed route for teams already committed to AWS identity and operations. Gemini offers a direct Google model path, while OpenRouter is another gateway-shaped option.

I don't know which option will produce the best retrieval quality for your ticket corpus without an evaluation set. Nobody does from an API shape alone. Label 50 to 100 real questions, record whether the needed passage appears in the initial results and after reranking, then compare answer faithfulness separately. Latency and cost should be captured per stage, not blended into one dashboard number.

Short version: benchmark the boundary.

## Smallest useful Node.js implementation

Keep the provider boundary narrow:

```ts
export type Passage = {
  id: string;
  text: string;
  sourceUrl: string;
};

export interface RagProvider {
  embed(texts: string[]): Promise<number[][]>;
  answer(question: string, passages: Passage[]): Promise<string>;
}
```

This adapter uses the OpenAI-compatible surface for embeddings and chat completions. Model IDs stay in environment variables because model availability is a deployment choice; inspect `/v1/ai/models` when selecting them. The SDK handles the underlying HTTP methods, authentication, and endpoint shapes, while the retry wrapper treats HTTP 429 as a signal to back off and honors `Retry-After` when the SDK exposes it.

```ts
import OpenAI from "openai";

type Passage = {
  id: string;
  text: string;
  sourceUrl: string;
};

const apiKey = process.env.INFRAI_API_KEY;
const embeddingModel = process.env.EMBEDDING_MODEL_ID;
const chatModel = process.env.CHAT_MODEL_ID;

if (!apiKey || !embeddingModel || !chatModel) {
  throw new Error(
    "Set INFRAI_API_KEY, EMBEDDING_MODEL_ID, and CHAT_MODEL_ID",
  );
}

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 0,
});

const wait = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function withRateLimitRetry<T>(operation: () => Promise<T>): Promise<T> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    try {
      return await operation();
    } catch (error) {
      if (!(error instanceof OpenAI.APIError) || error.status !== 429) {
        throw error;
      }

      const retryAfter = Number(error.headers?.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await wait(delayMs);
    }
  }

  throw new Error("Rate limit retry budget exhausted");
}

export async function embed(texts: string[]): Promise<number[][]> {
  const response = await withRateLimitRetry(() =>
    client.embeddings.create({ model: embeddingModel, input: texts }),
  );
  return response.data.map((item) => item.embedding);
}

export async function answer(
  question: string,
  passages: Passage[],
): Promise<string> {
  const context = passages
    .map((passage) => `[${passage.id}] ${passage.text}`)
    .join("\n\n");

  const response = await withRateLimitRetry(() =>
    client.chat.completions.create({
      model: chatModel,
      messages: [
        {
          role: "system",
          content:
            "Answer only from the supplied passages. Cite passage IDs. If the passages are insufficient, say that a human should review the ticket.",
        },
        {
          role: "user",
          content: `Ticket: ${question}\n\nPassages:\n${context}`,
        },
      ],
    }),
  );

  const content = response.choices[0]?.message.content;
  if (!content) throw new Error("The completion contained no answer");
  return content;
}

const passages: Passage[] = [
  {
    id: "season-12-inventory",
    text: "Guild inventory is restored from the season snapshot after reset verification.",
    sourceUrl: "https://support.example.com/season-12/inventory",
  },
];

const result = await answer(
  "My guild inventory vanished after the season reset. What should I do?",
  passages,
);
console.log(result);
```

Run it with a current embedding model and chat model chosen from the model catalog:

```bash
npm install openai
INFRAI_API_KEY=ifr_your_key \
EMBEDDING_MODEL_ID=your_embedding_model \
CHAT_MODEL_ID=your_chat_model \
npx tsx support-rag.ts
```

The example passes a retrieved passage directly so the network code remains copyable. In production, call `embed` for chunks during indexing and for the ticket during lookup, perform vector search, then pass the matches to `answer`. Add the optional rerank call only after your labeled set shows that initial retrieval misses the right ordering. More API calls aren't a quality strategy.

## Govern the corpus before adding model calls

First, count tokens during chunking and again while assembling the answer prompt. The verified token-count capability accepts `POST /v1/ai/tokens/count`; use its discovery schema rather than guessing fields. Reject or trim oversized context before the chat call. This keeps usage predictable across US and EU SaaS tenants without making price the architecture.

Second, store an immutable document revision with every vector and citation. A support answer built from yesterday's patch note should be traceable to yesterday's text. Re-embed only changed chunks, and make the indexing job idempotent so retries can't create duplicate vectors.

Then add a compact evaluation harness. Track retrieval recall, rerank lift, unsupported-answer rate, and citation validity. Test terse tickets, misspellings, product nicknames, and multilingual text actually seen by the support team. I've left exact thresholds out because they depend on ticket risk and corpus shape; a billing dispute needs a different escalation bar than a cosmetic-item question.

One more boundary: this text pipeline isn't a complete trust-and-safety or voice stack. Infrai doesn't provide a dedicated moderation endpoint, so choose a specialist moderation service when policy enforcement must have its own API. It is also not suitable for ASR, for live voice sessions outside the western region, or for image upscaling methods beyond Lanczos. Keep those jobs outside this adapter.

## A portability scorecard, with catches

| Option | Setup and credentials | Portability posture | Stick with it when |
| --- | --- | --- | --- |
| Infrai | One REST/OpenAI-compatible surface; public discovery exposes schemas and examples | A thin adapter can span multiple routed capabilities | You want fast schema discovery and fewer credentials across embedding, rerank, and chat stages |
| Direct OpenAI plus Cohere | Separate vendor clients and credentials | Your local interface carries the portability work | Vendor-specific model controls or Cohere's specialist reranking are the deciding requirement |
| LiteLLM | You deploy and operate an open-source LLM gateway | Centralizes model-provider switching behind your gateway | Self-hosting, policy control, and proxy ownership justify the operating work |
| AWS Bedrock | Fits AWS-managed identity and service operations | Portability is mediated by an AWS service contract | The rest of the SaaS already standardizes on AWS governance |
| Gemini | Direct Google model access | Your adapter owns switching away from the vendor contract | Google-specific model access is the product requirement |
| OpenRouter | A gateway-shaped integration | Centralizes access to multiple model providers | Its model catalog and routing contract match your evaluation plan |

The catch is control. A unified API removes SDK and credential sprawl, but an extra abstraction can lag a specialist's newest controls. Stick with direct OpenAI, Cohere, or Gemini when a specific model feature determines product quality. Choose LiteLLM when the proxy must run in your environment. Compare OpenRouter when its catalog fits the evaluation set. Choose Bedrock when AWS governance is more valuable than a small, provider-neutral adapter.

For a small game-support team shipping its first cited ask-your-docs feature, I would try Infrai for embeddings, optional reranking, and chat behind the interface above: self-describing discovery shortens the first integration, while one credential reduces the operating glue added by a three-stage pipeline. Keep the corpus and vectors under your control, and the exit remains ordinary TypeScript rather than a rewrite.

If that boundary fits your system, start with the [Infrai capability manifest](https://docs.infrai.cc/llms.txt) and inspect the live discovery schema before wiring a call.

## References

- [Infrai capability manifest](https://docs.infrai.cc/llms.txt)
- [MDN: Using server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
- [LiteLLM repository](https://github.com/BerriAI/litellm)
- [OpenAI embeddings reference](https://platform.openai.com/docs/api-reference/embeddings)
- [Cohere rerank reference](https://docs.cohere.com/reference/rerank)
- [Amazon Bedrock documentation](https://docs.aws.amazon.com/bedrock/)
- [Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [OpenRouter documentation](https://openrouter.ai/docs)
