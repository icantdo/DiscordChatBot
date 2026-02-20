# Luna — Discord Chatbot Architecture

> Closed-source project. This document covers architecture and design decisions only.

Luna is a Discord chatbot designed to simulate human-like conversation behavior. It tracks social relationships, acts autonomously when bored, and operates on a four-layer memory system. All text and personality are in Turkish.

---

## Overview

```
Discord Message
     │
     ▼
┌─────────────────────┐
│   3-Layer Filter    │
│  Hard → Interest → Rate │
└────────┬────────────┘
         │
         ▼
┌─────────────────────┐
│  Message Aggregator │  ← 2s debounce, per user
│  + Processing Queue │  ← FIFO, 0.5s delay
└────────┬────────────┘
         │
         ▼
┌─────────────────────────────────────────┐
│          Dual-Brain Architecture        │
│                                         │
│  Brain 1 (Orchestrator) — Decides      │
│       ↓                                 │
│  ActionRouter — Routes the decision    │
│       ↓                                 │
│  Brain 2 (BrainCore) — Generates reply │
└────────┬────────────────────────────────┘
         │
         ▼
┌─────────────────────┐
│   EmotionTagger     │  ← Extract emotion tags
│   + MediaManager    │  ← Select GIF
│   + ResponseScheduler│ ← Human typing simulation
└────────┬────────────┘
         │
         ▼
    Discord Channel
```

Independent background services also run continuously:
`AnalystAgent` · `BoredomEngine` · `GossipPipeline` · `Dreamer`

---

## Filter Layer

Every incoming message passes through three stages:

**Layer 1 — Hard Filters**
Bot messages, DMs, empty content, command prefixes (`.` `/` `$`), silence prefixes (`,` `//` `#`), link-only messages, and spam (last 2 messages per user check) are all eliminated here. Minimum character length is also enforced.

Note: Even messages that fail the hard filter are still stored to STM and LTM. This is required so the AnalystAgent can continuously build user profiles.

**Layer 2 — Interest Scoring (InterestScorer)**
A component-based scoring system. If the total score crosses a threshold, the message proceeds to processing:

| Component | Score |
|---|---|
| Bot mention / reply / name | 100 (guaranteed response) |
| Interest keyword match | +30 (per keyword) |
| Question mark / emotional expression | +10 / +5 |
| Long message (>100 chars) | +15 |
| VIP user | +20 |
| Focus mode bonus | +30 |
| Previously talked to this user | +10 |
| Chaos factor | +0-15 (random) |

**Layer 3 — Rate Limiter**
Normal users: 12/min · VIP users: 20/min

---

## Dual-Brain Architecture

### Brain 1 — Orchestrator (Decision Layer)

A small, fast model with no personality — cold and analytical. It responds to each incoming message with a structured decision:

- **What's the intent?** (intent + urgency + confidence)
- **What should happen?** → `CHAT` · `IGNORE` · `SEARCH` · `TOOL_USE`
- **How should it be written?** → Mood, temperature, token limit, system injection

The Orchestrator also dynamically injects context into Brain Core's system prompt: who sent the message, their relationship status with the bot, and which conversation mode to use.

Prompt injection protection is in place: dangerous patterns and non-whitelisted prefixes are blocked.

### ActionRouter (Routing Layer)

Takes the Orchestrator's decision and routes it to the appropriate handler:

- `SEARCH` → Keyword search in LTM, injects results into Brain Core
- `TOOL_USE` → Web search (Serper.dev), writes result to both Brain Core and memory
- `CHAT` → Passes directly to Brain Core
- Falls back to `CHAT` on any failure

When a web search is performed, results are framed as the bot's "absolute knowledge" before being sent to Brain Core.

### Brain 2 — BrainCore (Response Layer)

A larger, higher-quality model. The system prompt is dynamically assembled from these blocks:

```
[Identity] + [Mode] + [Date/Time] + [User Profile]
+ [Social Graph Context] + [Chaos Memory] + [Rules]
+ [Emotion Tag Instruction]
```

**Mode System**: `default` · `gaming` · `roast` · `sleepy`
Manual override takes priority, then content-based detection, then automatic time-of-day selection.

The emotion tag is embedded inside the response and extracted before cleanup. The method returns a `(text, gif_url)` pair.

---

## Four-Layer Memory

```
┌────────────────────────────────────────────────────┐
│                   Memory System                    │
│                                                    │
│  STM      →  immediate context (15 msg / 5 min)   │
│  LTM      →  SQLite + FTS5 (keyword search)       │
│  VectorDB →  ChromaDB + MiniLM (semantic search)  │
│  GraphDB  →  Neo4j (social relationship graph)    │
└────────────────────────────────────────────────────┘
```

**STM (Short-Term Memory)**
Last 15 messages per channel, 5-minute TTL. Provides immediate conversational context.

**LTM (Long-Term Memory)**
SQLite with FTS5 full-text search. Turkish unicode tokenizer. Entries below a confidence threshold (<0.3) are not stored. An echo filter prevents memory from bloating with repeated content.

**Vector Memory**
ChromaDB with `all-MiniLM-L6-v2` embeddings (384 dimensions). Finds results by semantic similarity — catches connections that keyword search misses. Cosine similarity, threshold 0.4.

**Graph Memory (Neo4j)**
Stores social relationships in a structured graph.

Node types: `User` · `Luna` · `Topic` · `Trait`

Edge types:

| Edge | Description |
|---|---|
| `RELATIONSHIP` (User→Luna) | Affinity score (-100 to +100) |
| `INTERACTED_WITH` (User↔User) | User-to-user sentiment (EMA blended) |
| `INTERESTED_IN` (User→Topic) | Interest area and intensity |
| `HAS_TRAIT` (User→Trait) | Personality trait and confidence score |
| `CHAOS_LINK` (Topic↔Topic) | Unexpected connections discovered by Dreamer |

LTM can operate on its own but hallucinates without support. All four sources together ensure consistency and suppress hallucination.

---

## AnalystAgent — User Profiling System

A periodic background analysis runs for each user:

1. Trigger: 50 new messages from a user
2. Last 50 messages from LTM are read (30-day window)
3. An LLM analyzes them and extracts:
   - Profile summary and general mood
   - Affinity delta (shift in the user's attitude toward the bot)
   - Topics and personality traits
4. Results are written to both the SQLite `user_profiles` table and Neo4j
5. The Orchestrator reads this profile as needed to enrich the system prompt

Drift limits: -10/+5 per analysis · ±15 per day. This prevents the bot from suddenly becoming too close or hostile toward someone.

---

## BoredomEngine — Dopamine Seeking System

The bot gets bored when there's no conversation. Each channel has a boredom meter from 0 to 100.

**Meter Mechanics**:

| Event | Change |
|---|---|
| 60s of silence | +2.0 |
| Short boring reply ("ok", "lol") | +5.0 |
| Bot gets mentioned | -25.0 |
| Normal message | -3.0 |
| Long message (>50 chars) | -2.0 (additional) |
| Chaotic action completed | -15.0 (dopamine hit) |

**3 Mood Tiers & 9 Action Types**:

| Mood | Range | Actions |
|---|---|---|
| Observer | 0-30% | Emoji reaction, Discord status change |
| Poking | 30-70% | Old memory dig, baiting, ghost ping |
| Chaos | 70-100% | Topic hijack, self-dialogue, hallucinated reply, channel rename |

Channel renames are automatically reverted after a set duration. All names are restored when the bot shuts down.

**VictimSelector** decides which user to target:
1. Bot owner (always first if online)
2. Most recently active user
3. Users with negative sentiment (social tension from graph)
4. User with the lowest affinity score
5. Random online user not recently active in the channel

---

## GossipPipeline — Social Detective

Automatically detects interactions between users and writes them to the Neo4j graph.

Triggers: replies, @mentions, back-to-back messages within 5 seconds (co-presence).

Messages are accumulated in batches (6 messages or 120 seconds, whichever comes first). An LLM analyzes the batch and extracts who interacted with whom, and with what sentiment (-1 to +1). The result is written to Neo4j as `INTERACTED_WITH` edges; old sentiment is EMA-blended (old×0.7 + new×0.3).

This is how the bot learns patterns like "these two always argue" or "those two get along."

---

## Dreamer — Chaos Link Discovery

A background service that runs every 6 hours. Its purpose: discover topics that are semantically distant from each other but connected in unexpectedly interesting ways.

**Algorithm (Semantic Drift)**:
1. All `Topic` nodes in the graph are embedded using `all-MiniLM-L6-v2`
2. KNN search is performed via the Neo4j vector index
3. Directly related topics (same user's interests) are filtered out
4. The "sweet spot" is captured: 0.65-0.85 similarity (homonyms, conceptual accidents)
5. A `CHAOS_LINK` edge is created between matched topic pairs

These links are injected into Brain Core's prompt as a `[CHAOS MEMORY]` block. The bot occasionally slips these unexpected associations into conversation.

---

## Emotion and Media System

**EmotionTagger**: Extracts the hidden emotion tag embedded in Brain Core's response. Captures both primary and secondary emotions, along with intensity.

**MediaManager**: Sends GIFs based on emotion intensity. Recently sent GIFs are not repeated (FIFO cache).

| Intensity | GIF Probability |
|---|---|
| High | 80-100% |
| Medium | 30% |
| Low | 0% |

**ResponseScheduler**: Simulates human typing speed. Base delay per character plus random variance. The GIF is always sent after the last sentence, as a separate "punchline" moment (0.5s typing indicator).

---

## System Parameters

| Parameter | Value |
|---|---|
| STM size | 15 messages / channel |
| STM TTL | 5 minutes |
| LTM retrieval limit | 5 results |
| Vector similarity threshold | 0.4 |
| Analyst trigger | 50 messages |
| Analyst refresh | 7 days (stale profile) |
| Focus mode duration | 180 seconds |
| Boredom heartbeat | 60 seconds |
| Boredom action cooldown | 180 seconds / channel |
| Channel rename duration | 300 seconds |
| GIF cooldown | Last 5 URLs tracked |
| Dreamer interval | 6 hours |
| Dreamer KNN similarity range | 0.65 - 0.85 |
| Web search results | 3 organic results |
| Orchestrator timeout | 10s (3 attempts) |

---

## Technology Stack

| Layer | Technology |
|---|---|
| Platform | Discord (discord.py) |
| LLM — Orchestrator & Boredom | Llama 3.1 8B Instruct |
| LLM — BrainCore | Grok 4.1 Fast |
| LLM — AnalystAgent | DeepSeek V3.2 |
| Embedding Model | all-MiniLM-L6-v2 (384 dim) |
| Relational DB | SQLite (WAL mode, FTS5) |
| Vector DB | ChromaDB |
| Graph DB | Neo4j |
| Web Search | Serper.dev |
| Language | Python (async/await) |
| Configuration | Pydantic + .env |

---

*This document serves as a general architectural reference. Implementation details remain closed source.*
