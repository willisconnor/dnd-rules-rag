# D&D Rules RAG — Project Guide

## What this is

A retrieval-augmented generation tool that answers D&D 5th edition rules questions by
retrieving from both the 2024 and 2014 System Reference Documents (SRD) and returning a
structured comparison: the 2024 rule, the 2014 rule (only when it actually differs), and
a clear "no equivalent" state when a mechanic is edition-specific.

The retrieval layer is the core engineering problem, not the LLM call. Getting a 2024
answer is easy. Correctly aligning it with its 2014 counterpart when the rule was renamed,
restructured, or split is the actual hard part, and the reason this project is worth doing.

## Source data and licensing

- Only ingest the official SRDs: SRD 5.2 (2024, CC-BY-4.0) and SRD 5.1 (2014, OGL).
- Never ingest full Player's Handbook/DMG text or any other copyrighted book content.
- No Reddit or other community-sourced content gets ingested into the retrieval corpus.
  If a "how tables actually rule this" feature ever comes back, it must be original
  human-written synthesis, not scraped or API-pulled community text, and must not be
  positioned as authoritative — SRD text is the only "rule," anything else is commentary.

## Architecture

Two backends, each doing what its language is actually good at, talking over REST:

```
/ingestion          Python — one-time/offline pipeline, portfolio-quality, not throwaway
  chunk.py           split raw SRD text by heading structure, not fixed token windows
  tag.py             LLM-assisted topic tagging (topic_key, edition, section_title)
  align.py           builds the topic_key -> {2024 chunk_ids, 2014 chunk_ids} map
  review_export.py   dumps tagged chunks to CSV/JSON for manual review
  config.py          model name, batch size, file paths — no hardcoded values in scripts
  tests/
    test_chunk.py    chunker must not split mid-sentence on real SRD files
    test_tag.py      topic_key normalizer must produce consistent slugs for near-dupes

/retrieval           Python — FastAPI service
  embed.py           query + corpus embedding (local model, same model for both)
  faiss_index.py     FAISS vector store build + load
  retrieve.py        hybrid retrieval: topic_key lookup first, semantic fallback second
  generate.py        prompt template + Ollama call, structured JSON output + parsing
  api/
    routes.py        POST /query -> structured response
  tests/

/data
  srd-2014/           raw + chunked
  srd-2024/           raw + chunked
  topic_map.json       the alignment table built by align.py

/frontend            TypeScript — Next.js (same stack as Lumière)
  app/
  components/
    RuleCard.tsx      renders 2024 rule / 2014 rule / no-equivalent states
  lib/
    api-client.ts     calls the FastAPI /query endpoint

/discord-bot         TypeScript — stretch goal, thin wrapper calling the same /query endpoint
```

## Key design decisions (don't relitigate these without a reason)

- **Generation model:** Ollama, local, Llama 3.1 8B Instruct as the default target
  (benchmark against Mistral 7B once retrieval is stable — don't tune the model before
  retrieval quality is proven, tuning generation on top of bad retrieval wastes time).
- **Taxonomy:** semi-automated. LLM tags each chunk with a topic_key during ingestion,
  fed the running list of already-used topic_keys so it reuses `grappling` instead of
  inventing `grapple_rules` on a later chunk. Human review pass is mandatory before the
  tagged data feeds the embedding step — this is the highest-leverage hour in the project,
  don't skip it.
- **Retrieval strategy:** hybrid. Try direct topic_key lookup across both editions first;
  fall back to semantic similarity when no topic_key match exists. Never force a 2014
  comparison onto a 2024-only mechanic — say clearly there's no equivalent.
- **Build order:** ingestion script -> manual review -> embedding/FAISS build -> retrieval
  function (tested standalone with known tricky queries) -> generation function (tested
  standalone) -> API endpoint (tested via Postman) -> frontend last. Don't jump ahead to
  frontend before retrieval is validated — that's where the real risk lives.
- **Ingestion script is portfolio code, not a throwaway.** It needs to be idempotent
  (safe to re-run without corrupting the taxonomy), have a dry-run/diff mode showing what
  changed since the last run, be config-driven, log a readable summary, and have tests.

## How to work through this project

Work through the build order below one stage at a time. Do not scaffold the entire repo
in one pass. After completing a stage, stop and report what was built, then wait for
explicit go-ahead before starting the next stage.

1. Ingestion pipeline (chunk, tag, align, review-export) + its tests.
2. **Stop here.** The manual review pass (human reads the tagged chunk export) happens
   outside of Claude Code entirely. Do not proceed past this point until told the review
   is done and the tagged data is finalized.
3. Embedding + FAISS index build, using the reviewed/finalized tagged data.
4. Retrieval function, tested standalone against the fixed set of known tricky queries
   (a renamed 2024 rule, a 2024-only mechanic, an unchanged rule) before moving on.
5. Generation function (prompt template + Ollama call), tested standalone.
6. API endpoint wrapping retrieval + generation, tested via Postman/curl.
7. Frontend, last.
8. Discord bot, stretch goal, only after the above is solid.

If asked to "just build the whole thing," push back and confirm this staged approach
still applies unless explicitly told to abandon it for this session.

## Engineering principles

1. Think Before Coding
Don't assume. Don't hide confusion. Surface tradeoffs.

Before implementing:

State your assumptions explicitly. If uncertain, ask.
If multiple interpretations exist, present them - don't pick silently.
If a simpler approach exists, say so. Push back when warranted.
If something is unclear, stop. Name what's confusing. Ask.
2. Simplicity First
Minimum code that solves the problem. Nothing speculative.

No features beyond what was asked.
No abstractions for single-use code.
No "flexibility" or "configurability" that wasn't requested.
No error handling for impossible scenarios.
If you write 200 lines and it could be 50, rewrite it.
Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

3. Surgical Changes
Touch only what you must. Clean up only your own mess.

When editing existing code:

Don't "improve" adjacent code, comments, or formatting.
Don't refactor things that aren't broken.
Match existing style, even if you'd do it differently.
If you notice unrelated dead code, mention it - don't delete it.
When your changes create orphans:

Remove imports/variables/functions that YOUR changes made unused.
Don't remove pre-existing dead code unless asked.
The test: Every changed line should trace directly to the user's request.

4. Goal-Driven Execution
Define success criteria. Loop until verified.

Transform tasks into verifiable goals:

"Add validation" → "Write tests for invalid inputs, then make them pass"
"Fix the bug" → "Write a test that reproduces it, then make it pass"
"Refactor X" → "Ensure tests pass before and after"
For multi-step tasks, state a brief plan:

1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

## Commands

<!-- fill in once scripts exist, e.g. -->
<!-- `python ingestion/chunk.py --edition 2024` -->
<!-- `python ingestion/tag.py --dry-run` -->
<!-- `uvicorn retrieval.api.routes:app --reload` -->
<!-- `npm run dev` (frontend) -->

## Testing philosophy

Test the chunker and taxonomy normalizer with real SRD text, not synthetic examples —
the failure modes that matter are specific to this actual corpus (nested lists, tables,
cross-references between sections). Test retrieval quality with a fixed set of known
tricky queries (a renamed 2024 rule, a 2024-only mechanic, an unchanged rule) and check
them by hand after any change to chunking, tagging, or the retrieval function.
