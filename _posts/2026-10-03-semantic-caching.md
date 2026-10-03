---
layout: post
title: "Teaching an LLM to say \"I've answered this before\""
description: "What I learned building a semantic caching middleware for LLM apps during my internship at Elyadata: embeddings, HNSW, cache invalidation, design patterns, observability and testing."
image: /assets/img/posts/semantic-caching/cover.png
tags: [llm, caching, system-design]
---

Well, I'm not used to publishing the work that I do or the stuff that I write. I kept it lowkey for A LONG TIME, but now here we are xD. So hello to anyone who's going to read this, hope you're doing well. If you're expecting a serious engineering blog that will change the way you look at the world, sorry to disappoint. I'm just writing down things I think are cool and giving the work I do a bit of value.

Speaking of value, this summer marked an interesting milestone in my career. I did an AI engineering internship at Elyadata, where I built a Python library that lets you put a semantic caching middleware between your users and the LLM you use. Lots of big words, I know, but I'll explain the whole thing bit by bit: the concepts I came across, the things I learned, the problems I ran into, and the state of enlightenment I reached at the end of the road.

## The problem: LLMs are expensive to ask twice

Let's start simple. LLMs are DEMANDING. Whenever someone prompts ChatGPT or Claude, they don't see how much computation happens under the hood or how much electricity it takes. You can look at this from the side of a company serving millions of users, or from the side of a company paying for every API call. Either way, a lot gets spent, and a good part of it goes to the same repetitive questions.

Say someone asks an LLM for the capital of France and gets "Paris". Then another user, on the other side of the world, asks the exact same question and gets the exact same answer. We computed the same answer twice, for a fact that won't change anytime soon.

When I started reading about this, I found that a lot of solutions already exist at different levels. Some live inside the model. KV caching, for example, stores intermediate attention results so the model doesn't redo work for tokens it has already processed while generating one answer. Others sit on top of the LLM application and avoid calling the model at all. My work is in the second group.

So the goal is clear: detect whether a question has been asked before, and if it has, serve the stored answer instead of generating a new one. Easy, or at least it sounds easy when you put it like that.

## Semantic caching in one picture

A normal cache only helps if the new request is identical to an old one, down to the last comma. People don't ask questions like that. "What's the capital of France?", "capital of france??" and "Which city is France's capital?" are the same question to a human and three different strings to a computer. A semantic cache compares what the questions mean.

On top of that, I had three requirements:

1. It has to be fast. Checking the cache adds a step in front of every request, so that step must cost very little compared to calling the LLM.
2. It has to be accurate. Serving a wrong answer is worse than regenerating a right one.
3. It has to respect privacy. A cache remembers things, so it must never store personal information (PII) like names, emails or card numbers.

This is the architecture I ended up with:

[![The full architecture: request plane, privacy and model services, cache and persistence, and operations](/assets/img/posts/semantic-caching/architecture.png)](/assets/img/posts/semantic-caching/architecture.png)

*Click the diagram to open it full size.*

This might sound cliché, but I don't care: this figure makes me genuinely happy. Starting from zero and ending up with something like that made me proud.

Here's how a request goes through it. The user's question gets turned into a vector by an embedding model: a list of numbers (1024 of them in my case) where questions with similar meanings end up close to each other. That vector gets compared to the vectors already stored in the cache. If one is close enough, we return its stored answer. If nothing is close enough, the LLM generates a new answer, we send it back, and we store the question and answer so the next similar question gets it instantly.

![Every request takes one of two paths: a hit returns a stored answer, a miss goes to the LLM and gets stored for next time](/assets/img/posts/semantic-caching/hit-vs-miss.png)

That's the whole idea in one paragraph. The rest of this post is about everything that paragraph hides.

## "Close enough" is doing a lot of work

To compare two questions, I compare their vectors with cosine similarity. It measures the angle between two vectors: 1 means they point in exactly the same direction, and the lower the number, the less related the questions are. "What's the capital of France?" and "Which city is France's capital?" land almost on top of each other even though they barely share any words.

Here's the trap. "What's the capital of France?" and "What's the capital of Germany?" are ALSO very close to each other, because they're the same sentence with one word changed.

![A strict threshold only lets true paraphrases hit; a loose one lets "capital of Germany" hit too and get "Paris"](/assets/img/posts/semantic-caching/similarity-threshold.png)

If the threshold is too loose, the person asking about Germany gets told "Paris" with full confidence. If it's too strict, almost nothing hits the cache and the whole system is just extra latency. The threshold is basically the soul of the system, so I didn't want to pick it by feel.

I benchmarked it on the Quora Question Pairs dataset, which has pairs of questions labelled by humans as duplicates or not. For each of 10,000 pairs, I cleared the cache, sent the first question, then sent the second one and checked whether it hit. A hit on a real duplicate is a true positive. A hit on two different questions is a false positive, which is exactly the "Germany gets Paris" case. I ran this for every threshold from 0.50 to 0.95.

![Precision, recall and F1 across similarity thresholds, with the best F1 at 0.80](/assets/img/posts/semantic-caching/threshold-results.png)

Two curves matter here. Precision is "when the cache answers, how often is it right?" and recall is "of all the real duplicates, how many did the cache catch?". Raising the threshold makes the cache pickier: precision goes up and recall goes down. At 0.95, precision is about 88%, but the cache catches only 19% of real duplicates. At 0.50, it catches 65% of them but is wrong more often than it's right.

F1 combines the two into one score, and it peaks at a threshold of 0.80, with an F1 of 68.3% (precision 63%, recall 74%). That's the value all my production caches use. It's a trade-off, and depending on how costly a wrong answer is for your use case, you could reasonably pick 0.85 instead: fewer catches, but fewer mistakes.

## Don't cache people's secrets

Before a query gets anywhere near the cache, it goes through Microsoft Presidio, a library that detects PII in text. It comes with built-in recognizers for things like names, emails, phone numbers, credit cards, IBANs and IP addresses, and I added custom ones written in YAML, like an employee ID pattern (`EMP-` followed by five digits). Admins can upload new recognizer files through the API, and the analyzer reloads them without a restart.

If the query is clean, it goes down the normal path. If it contains sensitive data, the cache is skipped completely: no embedding, no lookup, nothing stored. The LLM still gets the original query so the user still gets a proper answer.

One honest caveat: this only protects what the cache stores. The LLM provider still sees the original question. If that matters for your use case, you need a provider you trust, or one you host yourself.

## How HNSW finds the nearest question

The cached vectors live in Redis Stack, and the search uses an index called HNSW (Hierarchical Navigable Small World). This part fascinated me, so let me explain it properly.

The naive way to find the closest stored question is to compare the new vector with every single stored vector. Each comparison is a dot product over 1024 numbers, and you'd do that for every entry in the cache, for every request. That works for a hundred entries and becomes a problem for a million.

HNSW avoids that by turning the stored vectors into a graph. Each vector is a node, connected to a handful of its nearest neighbours. To search, you start somewhere and keep jumping to whichever neighbour is closer to your query, until no neighbour is closer than where you already are. That greedy walk is the "navigable small world" part, and it comes from the same idea as six degrees of separation: in the right kind of graph, any two nodes are only a few hops apart.

![HNSW search: start in a sparse top layer with long jumps, then drop down to denser layers for finer steps](/assets/img/posts/semantic-caching/hnsw.png)

The "hierarchical" part is the clever bit. HNSW builds several layers. The bottom layer contains every vector. Each layer above contains a random, smaller subset, so the top layer has only a few nodes with long-distance links. A search starts at the top, makes big jumps to get to the right region fast, then drops down a layer and makes smaller, more precise jumps, and so on until the bottom. If you know skip lists, it's the same idea applied to a graph. The cost of a search grows roughly with the logarithm of the number of entries, while brute force grows linearly with it.

Two parameters control how the graph is built, and these are the values I used:

- `M = 16` is the maximum number of links each node keeps per layer. More links make the graph better connected and the search more accurate, at the cost of memory.
- `EF_CONSTRUCTION = 200` is how many candidate neighbours the algorithm considers when it inserts a new node. A higher value builds a better graph but makes inserts slower.

The catch is in the name of the problem it solves: approximate nearest neighbour search. HNSW can occasionally miss the true closest vector. For a cache, that's an easy trade to accept, since the worst case is a miss that sends the question to the LLM.

In my code, the search asks Redis for the single nearest entry (`KNN 1`) inside the cache's namespace. Redis returns a cosine distance, which is 1 minus the similarity, so a threshold of 0.80 becomes "accept the match if its distance is at most 0.20".

## Wait, this is a system design problem

Somewhere in the middle of the internship, I realised that the "AI" part of the project (embeddings and similarity) was maybe a third of the work. The rest was the kind of thing system design interviews are made of: where data lives, how long it stays valid, what happens when storage fills up, and what happens when the machine restarts. Caching is one of the oldest ideas in computing, and every one of its classic problems showed up in mine.

The pattern I followed has a name: cache-aside. The application checks the cache first. On a miss, it goes to the slow source (the LLM here), then writes the result into the cache itself. The cache never calls the LLM on its own. It's simple, and it puts every hard decision in your hands.

### Two tiers: fast memory and durable disk

![Redis handles every lookup in memory; Postgres keeps a durable copy; on startup, Redis is pre-warmed from Postgres](/assets/img/posts/semantic-caching/two-tiers.png)

Redis keeps the vectors in memory and answers every lookup. Memory is fast but forgetful, so every new entry is also written to PostgreSQL with the pgvector extension. Postgres keeps the full history: each entry's question, answer, embedding, hit count, timestamps, and whether it was later invalidated or evicted, and by which strategy.

Having two copies of the same data raises the question every distributed system eventually asks: what happens when they disagree? Two decisions in my code deal with that. When an entry is invalidated or evicted, the Postgres row is marked first and the Redis key is deleted second. If the service dies between the two steps, the worst case is a stale key in Redis that the next maintenance pass removes, instead of a deleted entry coming back from Postgres after a restart. And a failed Postgres write gets logged without failing the request, because the user's answer shouldn't depend on the archive being available.

### Hydration: don't start cold

An empty cache means every request is a miss, so the first minutes after a deploy would be the slowest and most expensive of the day. To avoid that, the service hydrates Redis when it starts: each cache loads its 10 most-hit entries from Postgres before it receives traffic.

The hydration query skips entries that were invalidated or evicted, and it only loads entries created with the same embedding model and the same vector dimension as the current one. That last filter is easy to forget and important. Vectors from two different embedding models live in two unrelated spaces, so comparing them gives meaningless similarity scores. If you switch models, the old cache has to go.

### Invalidation: when an answer stops being true

There's an old joke that there are only two hard things in computer science: cache invalidation and naming things. I get the joke now. "The capital of France" can stay cached for years, while "what's the weather today" shouldn't survive the night. An answer that was right when it was stored can quietly become wrong, and the cache will keep serving it with total confidence.

I implemented three invalidation strategies, each checked by a background loop:

- TTL (time to live): an entry expires once it reaches a fixed age, no matter how popular it is.
- Inactivity: an entry expires if nobody has asked for it in a while. Every hit resets its clock.
- Hit count: on each sweep, entries with fewer than 2 hits get removed. A new entry starts at 1, so it survives only if someone asks something similar before the next sweep. This one cleans out one-off questions and keeps the ones people actually repeat.

### Eviction: when the cache is full

Invalidation removes what's wrong. Eviction removes what's least worth keeping once the cache passes its size limit. They sound similar, and I mixed them up at first, but they answer different questions.

![FIFO evicts the oldest entry, LRU the one unused for longest, LFU the one used least often](/assets/img/posts/semantic-caching/eviction-policies.png)

I implemented the three classic policies. FIFO (first in, first out) evicts the oldest entries. It's simple, but it ignores how useful an entry is. LRU (least recently used) evicts whatever hasn't been asked for in the longest time, which works well when recent questions tend to come back. LFU (least frequently used) evicts the entries with the fewest hits, which protects the evergreen questions that keep coming up.

To compare all of these side by side, the service runs six independent caches, one per policy: `ttl`, `inactivity` and `hit_count` for invalidation, and `fifo_eviction`, `lfu_eviction` and `lru_eviction` for eviction. Each has its own Redis prefix, its own namespace and its own API route, so you can send the same traffic to all of them and see which one behaves best.

## My first time using design patterns (for real)

I'd seen design patterns in class, and honestly they felt like vocabulary for exams. This was the first project where I reached for them because the code needed them, and they made it much easier to change and to test. I used four.

![A Façade in front, Strategies for invalidation and eviction, factory methods for clients, and shared single instances](/assets/img/posts/semantic-caching/design-patterns.png)

### Strategy

FIFO, LRU and LFU all do one job, picking which entries to drop, in three different ways. Instead of one big `if policy == "lru": ... elif ...` block, each policy is its own class behind a shared abstract interface:

```python
class CacheEvictionStrategy(ABC):
    @abstractmethod
    async def select_entries_for_eviction(
        self, entries: list[CacheEntry], number_to_evict: int
    ) -> list[CacheEntry]: ...


class LRUEvictionStrategy(CacheEvictionStrategy):
    async def select_entries_for_eviction(self, entries, number_to_evict):
        if number_to_evict <= 0:
            return []
        sorted_entries = sorted(
            entries,
            key=lambda e: e.last_accessed_at or e.created_at,
        )
        return sorted_entries[:number_to_evict]
```

The cache receives a strategy object when it's created and calls `select_entries_for_eviction` without knowing which policy it holds. The three invalidation strategies work the same way through a `should_invalidate_entry(entry)` method. Adding a new policy means writing one new class, and nothing else in the cache changes.

### Factory

Each part of the configuration knows how to build the client it describes. The Redis settings have a `get_client()` method that returns a configured Redis connection, the Ollama settings do the same, and the Postgres settings have a `get_service()` that creates the connection pool and initializes it. The rest of the code asks for a client and never deals with hosts, ports or passwords. I used the same idea in the tests, where a pytest fixture returns a function that builds a cache with whatever threshold the test needs.

### Singleton

Some objects should exist exactly once per process. The settings object is created once at module level, and every file imports that same instance. At startup, I create one Redis client and pass it to all seven Redis services, one Postgres pool, one Presidio query processor and one embedding client, and the six caches share them. Opening a new connection per cache would waste resources for no benefit.

It's also the pattern I'd be most careful with next time. Shared global state is convenient until you write tests, because every test ends up touching the same object. Passing these instances in explicitly, as I did for the caches, keeps that under control.

### Façade

Behind the scenes, a single request touches Presidio, the embedding model, Redis, Postgres and the LLM, and updates a handful of metrics on the way. Someone using the library shouldn't need to know any of that. The `SemanticCaching` class gives them one method, `get_answers(query)`, which returns the answer along with whether it was a cache hit. That's the Façade, and it's what turned a pile of components into a library someone else could actually use.

## Watching it work: Prometheus and Grafana

A cache you can't measure is a cache you can't trust. Is the hit rate 5% or 50%? Is the lookup fast, or is the embedding call slowing everything down? Did eviction run last night? Without numbers, you're guessing.

![The API and LiteLLM expose /metrics, Prometheus scrapes them every 5 seconds, and Grafana queries Prometheus](/assets/img/posts/semantic-caching/observability.png)

Prometheus works on a pull model. My FastAPI service exposes an HTTP endpoint, `/metrics`, that lists the current value of every metric in plain text. Every 5 seconds, Prometheus scrapes that endpoint (and LiteLLM's, which exposes its own metrics about model calls), stores each value with a timestamp, and builds a time series out of it. The application doesn't push anything or know Prometheus exists. It only keeps its counters up to date.

The Python client gives you three main metric types, and I used all three:

- A Counter only goes up. I use counters for requests, hits, misses, PII bypasses, invalidations and evictions, for example `semantic_cache_hits_total`.
- A Histogram records how long something took by sorting each measurement into buckets, which lets you compute percentiles later. I time the embedding call and the vector lookup separately, plus the latency of every HTTP response.
- A Gauge can go up and down. `semantic_cache_size` tracks how many entries each cache holds.

Every metric has a `cache` label (and the invalidation and eviction counters also have a `strategy` label), so one metric covers all six caches and they can be compared directly.

Counters on their own aren't very readable, since they only ever grow. The useful numbers come from queries in PromQL, Prometheus's query language. The hit rate per cache over the last five minutes is hits per second divided by requests per second:

```text
sum by (cache) (rate(semantic_cache_hits_total[5m]))
  /
sum by (cache) (rate(semantic_cache_requests_total[5m]))
```

And the 95th percentile of lookup time, built from the histogram buckets:

```text
histogram_quantile(0.95,
  sum by (le, cache) (rate(semantic_cache_lookup_latency_seconds_bucket[5m])))
```

Grafana sits on top of Prometheus. Each Grafana panel is a PromQL query like these, drawn as a graph that updates live, so you can watch the hit rate climb as the cache warms up, or catch a latency spike right when it happens.

## Testing it with pytest

Testing this system was its own puzzle. The real pipeline needs Redis, Postgres, Presidio, LiteLLM and an Ollama model running, and a real LLM answers differently every time. You can't write `assert answer == "Paris"` against that. So I split the tests in two.

![Unit tests swap the real services for fakes; integration tests run against the live Docker stack](/assets/img/posts/semantic-caching/testing.png)

The integration tests (23 of them) run against the full Docker stack. They start by checking that each service answers at all: Redis, Postgres, Presidio, the LLM and the embedding model. Then they run the real pipeline end to end: a miss followed by a hit, a PII query that has to bypass the cache, an entry that has to land in Postgres, hydration loading entries back from Postgres, and every invalidation and eviction strategy running against a real Redis.

The unit tests run with no external services at all, thanks to fake components. This is where the abstractions from the design patterns section paid off. `SemanticCaching` only depends on interfaces (`LLM`, `Embedder`, `VectorStore`), so in tests I can hand it fakes that behave exactly how I want.

The fake embedder is my favourite. It never calls a model. It maps a few words to hand-picked 3-dimensional vectors:

```python
embedding_map = {
    "hello":  np.array([1.0, 0.0, 0.0]),
    "hi":     np.array([0.95, 0.05, 0.01]),
    "hey":    np.array([0.94, 0.06, 0.02]),
    "baha":   np.array([0.0, 1.0, 0.0]),
    "random": np.array([0.3, np.sqrt(0.91), 0.0]),
}
```

Because I chose the vectors, I know the exact similarities: "hello" and "hi" have a cosine similarity of about 0.998, and "hello" and "random" have exactly 0.3. The fake LLM returns fixed answers and counts how many times it was called. That counter is the trick: if `llm.calls` didn't go up, the answer came from the cache.

With those two fakes, testing the threshold becomes precise. A pytest fixture builds a cache with whatever threshold a test needs, and the tests read almost like the threshold section of this post:

```python
@pytest.mark.asyncio
async def test_false_cache_hit(semantic_layer_factory):
    """Very permissive threshold: unrelated words hit"""
    semantic_layer = semantic_layer_factory(threshold=0.1)
    await semantic_layer.get_answers("hello")
    assert semantic_layer.llm.calls == 1
    await semantic_layer.get_answers("random")
    assert semantic_layer.llm.calls == 1  # false hit


@pytest.mark.asyncio
async def test_better_threshold(semantic_layer_factory):
    """Selective threshold: unrelated words miss"""
    semantic_layer = semantic_layer_factory(threshold=0.9)
    await semantic_layer.get_answers("hello")
    assert semantic_layer.llm.calls == 1
    await semantic_layer.get_answers("random")
    assert semantic_layer.llm.calls == 2  # went to the LLM
```

The other unit tests cover the invalidation strategies (an entry created 100 seconds ago with a 60-second TTL must be invalidated, and a TTL of zero must raise an error), the Presidio query processing, and the admin authentication. Since everything in the pipeline is async, the tests use `pytest-asyncio`. A `pytest.ini` at the root tells pytest where to look, so running `pytest` alone finds all of them, and `pytest testing/unit_tests/` runs only the fast ones.

## Did it actually help?

The threshold benchmark measured accuracy. To measure speed, I replayed 100 real user queries from the LMSYS-Chat-1M dataset (a public collection of real conversations people had with chatbots) against three of the caches and recorded every response time.

![Average response time on a cache hit versus a miss for the TTL, inactivity and hit-count caches](/assets/img/posts/semantic-caching/latency-results.png)

| Cache | Hit rate | Average hit | Average miss | Speed-up on a hit |
|---|---:|---:|---:|---:|
| TTL | 19% | 1.5 s | 47.8 s | 32× |
| Inactivity | 19% | 1.8 s | 74.6 s | 41× |
| Hit count | 33% | 1.6 s | 23.0 s | 15× |

On a hit, the answer came back in about 1.5 to 1.8 seconds. On a miss, it took between 23 and 75 seconds on average, because the model (a small Qwen model) was running locally through Ollama on my machine. A hit isn't instant either, since it still pays for the Presidio check and the embedding call before the lookup.

The hit-count cache had the best hit rate, 33%, meaning one question in three was answered without touching the LLM. On these 100 queries, the hits saved an estimated 8 to 16 minutes of waiting per cache.

These numbers come from a small local run, so take them as an order of magnitude more than a promise. On a faster model the gap between a hit and a miss would shrink, but the cost saved per hit (a full LLM call) stays.

## What's still rough

A few things I'd fix with more time:

- The startup code builds every Redis index with a hardcoded dimension of 1024 instead of reading it from the settings, so changing the embedding model means changing code in two places.
- Sensitive queries are supposed to leave a redacted record in Postgres for observability. A bug passes the redacted string where an entry object is expected, so that record is never saved. Nothing sensitive leaks, but the bypass isn't logged either.
- There are Grafana and Prometheus containers, but no dashboard is saved in the repo, so a fresh setup starts with an empty Grafana.
- A single threshold for every cache is a blunt tool. Some kinds of questions can tolerate a looser match than others, and I'd like to try per-namespace thresholds.

## Wrapping up

When I started, "semantic caching" was two big words on a task description. By the end, it was a system with a privacy layer, a vector index I actually understand, two storage tiers, six caching policies, metrics, and tests that don't need a GPU to run. The biggest lesson was that the AI part was the smallest part. Most of the work was deciding when "close enough" really is close enough, and what to forget.

Thanks for reading this far, seriously. If you've built something similar, or you think I got something wrong, I'd love to hear about it.
