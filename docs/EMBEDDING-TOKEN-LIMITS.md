# Understanding Embedding Model Token Limits

## The Problem We Discovered

When ingesting books into Alexandria, we noticed search quality degraded for longer text chunks. Queries that should have matched content in the middle or end of chunks were returning poor results.

After systematic debugging, we discovered the root cause: **nomic-embed-text has an effective 512-token limit**, despite documentation claiming 8192-token context support.

---

## Why BERT-Based Models Are Limited to 512 Tokens

nomic-embed-text is built on BERT architecture. BERT uses **learned positional embeddings** - during training, the model learns a unique embedding vector for each position (0, 1, 2, ... 511).

```
Position 0   → [0.12, -0.34, 0.56, ...]  (learned during training)
Position 1   → [0.23, -0.45, 0.67, ...]  (learned during training)
...
Position 511 → [0.89, -0.12, 0.34, ...]  (learned during training)
Position 512 → ???  (no learned embedding exists)
```

The original BERT was trained with positions 0-511 hardcoded. The model literally doesn't know what "position 513" means - it has no learned embedding for it.

**What happens when you pass >512 tokens:**
- The model either **silently truncates** (chops off everything after token 512)
- Or **errors out** (refuses to process)

In our case, it was silent truncation - the worst kind of failure because it looks like it's working.

---

## Why Nomic Claims 8192 Token Support

Nomic likely uses a technique like:

1. **Chunking** - Split the input into 512-token windows
2. **Embed separately** - Generate an embedding for each chunk
3. **Pool/Average** - Combine the chunk embeddings into one vector

This gives you *an* embedding for 8192 tokens, but it's fundamentally different from the model "understanding" 8192 tokens together.

**The critical limitation:** Semantic relationships between distant tokens are lost.

```
Token at position 100: "The contract expires on"
Token at position 4000: "December 31st, 2025"
```

With true 8192-token context, the model understands these are related. With chunked-and-averaged, each chunk is processed independently - the connection is severed.

---

## How This Affected Alexandria

### The Symptom
- Book chunks were created at ~1000-2000 tokens (seemed reasonable for semantic coherence)
- Search queries returned relevant results for content in the first ~500 tokens
- Content appearing later in chunks was effectively invisible to search

### The Debugging Process

1. **Noticed pattern:** "Why does search find chapter intros but not the meat of the content?"
2. **Hypothesis:** Embeddings aren't capturing full chunk content
3. **Test:** Searched for phrases that appeared at different positions in chunks
4. **Result:** Strong matches for early content, weak/no matches for late content
5. **Research:** Found the BERT 512-token architectural limit
6. **Verification:** Checked Nomic's technical report - confirmed BERT base architecture

### The Fix
Reduced chunk size to ~400 tokens with 50-token overlap:

```python
CHUNK_SIZE = 400       # Stay well under 512 limit
CHUNK_OVERLAP = 50     # Preserve context across chunk boundaries
```

This ensures:
- Every chunk fits entirely within the model's actual context window
- No content is silently truncated
- Overlap preserves semantic continuity between chunks

---

## Lessons Learned

1. **Marketing vs. Reality:** "8192-token context" can mean different things. Always verify with architectural details.

2. **Silent failures are dangerous:** The model happily accepted long inputs and returned embeddings. It just quietly ignored most of the content.

3. **Test at the boundaries:** When something "should work" but search quality is poor, test with content at different positions to isolate where the model's understanding breaks down.

4. **Read the technical report, not just the README:** The Nomic technical report mentions BERT architecture. That's the clue that 512 tokens is the real limit.

---

## References

- [Nomic Embed Technical Report](https://static.nomic.ai/reports/2024_Nomic_Embed_Text_Technical_Report.pdf) - Mentions BERT base architecture
- [HuggingFace Discussion: "What is the actual context size?"](https://huggingface.co/nomic-ai/nomic-embed-text-v1.5/discussions/51) - Community confusion about the limit
- [Original BERT Paper](https://arxiv.org/abs/1810.04805) - Describes the 512-position limitation

---

*Documented after debugging Alexandria's book ingestion pipeline, January 2026*
