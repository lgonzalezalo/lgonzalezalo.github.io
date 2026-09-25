---
layout: post
title: "Designing and benchmarking an intent classifier for e-commerce search"
date: 2026-09-25
categories: [machine-learning, engineering]
tags: [nlp, llm, embeddings, search, python]
---

> **Note (added after publication):** this analysis and benchmark were
> completed before TypeSafe AI's [Jev](https://www.typesafe.ai) — a
> proprietary "System One" decision model — attracted a great deal of
> attention following its launch on September 15, 2026. Jev addresses
> exactly the problem discussed here (typed, calibrated decisions
> instead of free-text generation), and it is, conceptually, the model
> most similar to Laya among the three approaches benchmarked below. I
> was not aware of it at the time this work was done, so it is not part
> of the comparison — see "Next steps" for where it would fit.

I am sharing the design and validation of an intent classification and
routing system for e-commerce searches that return no results — a
common situation in any store with a broad catalog, and a problem that
turns out to be more interesting than it first appears once it is
treated as a genuine routing problem rather than a generic error page.

Full code available on [GitHub](https://github.com/lgonzalezalo/zero-results-classifier).

## The problem

When a search returns no products, most search engines default to the
same "no results" page for everyone. But not every zero-result search
has the same cause: someone typing *"how long does shipping to my house
take"* is not searching for a product at all — they are asking a
customer service question that a product catalog, by design, can never
answer.

The idea is to detect that intent and redirect to the correct help page
(shipping, store hours, contact support, order status) instead of
leaving the user without an answer.

## Architecture: a two-layer cascade

```
                    zero-result query
                            │
                            ▼
              ┌─────────────────────────┐
              │  Layer 1: regex rules    │  ← microseconds, no cost
              └─────────────────────────┘
                            │
                  confident enough?
                    │             │
                   yes            no
                    │             │
                    ▼             ▼
              category       ┌─────────────────────────┐
              + action       │  Layer 2: semantic       │
                             │  classifier               │
                             └─────────────────────────┘
                                       │
                                       ▼
                             category + action,
                             or "unclassified"
```

**Layer 1** consists of weighted regex patterns per category — fast,
deterministic, and sufficient to resolve the unambiguous cases without
calling any model. On the validation dataset, this layer alone resolves
between 29% and 55% of traffic, depending on how much linguistic
variety the sample contains.

**Layer 2** is only triggered when Layer 1 does not reach a confidence
threshold. This is the part I was most interested in exploring: instead
of committing to a single architecture, I implemented three behind the
same interface, so they could be compared under identical conditions.

## Validation methodology

I built a dataset of 982 examples, combining template-generated
linguistic variants with a sample of synthetic "product noise" (invented
item names, simulating real traffic that does not belong to any
customer service category).

The debugging process turned out to be as revealing as the final
result: expanding the dataset from 20 to around 1,000 examples caused
the apparent accuracy to drop noticeably — not because the system had
become worse, but because the small sample had been hiding systematic
errors. Separating **dataset bugs** (poorly built templates that
guaranteed a collision with a rule) from **genuine taxonomy ambiguity**
(phrases that could legitimately belong to two categories) proved to be
the most valuable part of the whole process — a lesson that applies to
any classification project, not only to this one.

## Benchmark: three architectures for Layer 2

Measured on the same dataset, on the same machine, with per-query
latency recorded in real time. All three were evaluated **zero-shot**,
with no fine-tuning, in order to isolate what each architecture
contributes before investing in training anything:

| Model | Accuracy | p50 latency | p95 latency |
|---|---|---|---|
| **Qwen3:8b** (generative LLM, local via Ollama) | 96.4% | 848ms | 894ms |
| **bge-m3** (embeddings + cosine similarity) | 91.0% | 24ms | 29ms |
| **Laya-multilingual** (decision model, zero-shot) | 82.5% | 18ms | 20ms |

Two implementation details made a real difference for the generative
LLM:
- Forcing `format: json` on the Ollama call, together with `think: false`
  to disable Qwen3's extended reasoning. Without this, the reasoning
  block appeared before the JSON output and broke the parsing.
- Providing the prompt with each category's **description**, not only
  its name. With only the identifier (`store_info`), the model
  systematically failed to recognise less obvious vocabulary; adding a
  single line of context made this entire category of errors nearly
  disappear.

## The real design decision

Accuracy versus latency is not a secondary detail in a routing system —
it is the central decision, and it depends on where the business
constraint lies, not on which model is "better" in the abstract:

- If classification can run **asynchronously** (without blocking the
  response to the user, resolving in the background), 848ms is
  perfectly acceptable, and the generative LLM provides the best
  accuracy with no training effort at all.
- If classification ever needs to move to the **synchronous** response
  path, embeddings become the only viable option of the three, trading
  around 5 points of accuracy for roughly 35 times lower latency.

## Design takeaway

An intent router of this kind must assume, before classifying anything,
that a non-trivial share of the traffic reaching it is not real user
intent at all — noise, automated traffic, or simply text that does not
fit any meaningful category. A robust router in production requires an
upstream layer capable of discarding that traffic *before* attempting to
route it. This project deliberately focuses on intent routing itself;
filtering out illegitimate traffic is, by design, a separate and
upstream responsibility.

## Next steps

The most interesting combination I have not implemented yet is a
cascade *within* Layer 2 itself: embeddings first, since they are
inexpensive, escalating to the generative LLM only when similarity falls
into a genuinely uncertain range. This would recover much of the lost
accuracy without paying the LLM's cost on most queries.

Given the note above, benchmarking **Jev** alongside these three models
is now the obvious next addition. It addresses exactly the same "typed,
calibrated decision" space as Laya, coming from a much better-funded and
more actively developed company. Whether its calibrated-by-design
confidence remains reliable on a task this simple — the same question I
asked of Laya — is something worth testing rather than assuming,
particularly since it is currently proprietary, offered through a
waitlist, and API-only: a meaningfully different deployment trade-off
compared with the two self-hosted, open-weight options benchmarked here.

---

Full code, validation dataset, and reproducible benchmark available at
[github.com/lgonzalezalo/zero-results-classifier](https://github.com/lgonzalezalo/zero-results-classifier).
