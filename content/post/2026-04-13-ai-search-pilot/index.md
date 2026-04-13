---
title: "Can AI find what keyword search can't? A pilot on Dutch policy documents"
draft: false
date: 2026-04-13
tags: ["nlp", "search", "machine-learning", "python", "dutch"]
summary: "We built an AI-powered search engine to help PBL researchers find agricultural subsidy documents buried in hundreds of thousands of Dutch policy publications. Here's what worked, what didn't, and what surprised us."
---

**TL;DR:** We built an AI-powered semantic search engine on 200k+ Dutch government policy documents to help PBL researchers find subsidy documents related to nitrogen reduction. AI-only search reached up to 80% relevance. Hybrid search—contrary to what the literature suggests—underperformed. Here's the story of why.

---

## The problem with "agricultural subsidies"

Imagine you're a policy researcher at PBL (Planbureau voor de Leefomgeving), trying to understand which subsidies Dutch provinces and water authorities have granted to reduce nitrogen emissions. You turn to the government's official publication database—*Officiële Bekendmakingen*—and type "agricultural subsidies" into the search bar.

Too few results. The documents you're looking for use different words. You try just "subsidies." Now you're drowning in irrelevant results. There's no middle ground.

This is the classic failure mode of keyword search: it matches exact words, not meaning. A subsidy scheme for relocating livestock farms doesn't mention "agricultural subsidy" in its title, but a domain expert knows immediately that it's relevant. Keyword search doesn't understand that.

This is what motivated a pilot project by WLV and IBL at PBL: could an AI-powered search engine bridge that gap?

---

## The dataset: 200,000 policy documents

The *Officiële Bekendmakingen* portal publishes official government decisions, regulations, and subsidy schemes from municipalities, provinces, water authorities, and national government. For this pilot, we scoped to two publication types from 2015 to 2023:

- **Waterschapsblad** (regional water authority publications): ~117,000 documents
- **Provinciaal blad** (provincial publications): ~90,000 documents

That's a lot of documents. Each one is structured XML with headers, introductions, policy text, and standardized closings. Documents vary considerably in length—on average around 400 words for Waterschapsblad and 550 for Provinciaal blad—which turned out to matter a lot for how we split them up.

---

## What "AI search" actually means here

Before diving into the experiment, it's worth clarifying what makes this search "AI-powered." The key difference from keyword search is how documents and queries are represented.

In keyword search, a document is essentially a bag of words. In semantic search, each document is converted into a dense vector—a list of hundreds of numbers—by a *text representation model*. These numbers capture meaning: documents about related topics end up close together in this high-dimensional space, even if they share no exact words. When you search, your query is converted into the same kind of vector, and the engine retrieves the documents whose vectors are closest.

We tested three such models:

| | Model 1 | Model 2 | Model 3 |
|---|---|---|---|
| **Name** | robbert-2022-dutch-sentence-transformers | distiluse-base-multilingual-cased-v1 | BGE-M3 |
| **Language focus** | 100% Dutch | 15 languages incl. Dutch | 100+ languages incl. Dutch |
| **Parameters** | 119M | 135M | 567M |
| **Max sequence length** | 128 tokens | 128 tokens | 8192 tokens |

Model 1 was purpose-built for Dutch. Model 3 is a heavyweight multilingual model with a much longer context window. Model 2 sits somewhere in between.

---

## The experiment: one query, twelve configurations

We kept the experiment focused: one query, twelve configurations, evaluated on the top 5 results.

The query was: *"Voor welke activiteiten ter reductie van stikstof kan subsidie worden verstrekt?"* (Which activities to reduce nitrogen emissions can receive subsidies?)

The twelve configurations came from crossing three variables:
- **Text representation model**: Model 1, 2, or 3
- **Search mode**: AI-only (semantic) or hybrid (semantic + keyword)
- **Result filter**: with or without restricting to the category *Landbouw Organisatie en beleid*

A domain expert at PBL labeled all 60 resulting documents (5 results × 12 configurations) as relevant, partly relevant, or not relevant.

---

## The results: surprises all around

### Bigger isn't always better

Model 3 is nearly five times larger than Model 1, yet they perform similarly on this task. Model 2, despite having access to more multilingual training data, actually underperforms both. This is a useful reminder: model size and breadth don't guarantee better performance on a narrow, domain-specific task. A model trained specifically on Dutch text can outpunch its weight class.

### Hybrid search underperforms—and we're not sure why

This is the most counterintuitive finding. The literature consistently shows that hybrid search (combining semantic and keyword signals) outperforms semantic-only search. We found the opposite: hybrid configurations performed worse, and in some cases dramatically so. Models 1 and 3 in hybrid mode returned no relevant documents at all in the top 5 when no category filter was applied.

The frustrating part: we can't fully explain it. The hybrid search implementation runs on Azure AI Search, whose code isn't open source. We can't inspect how it's merging the two signals, or whether there's a hyperparameter we should have tuned. This is a limitation worth flagging for anyone else running similar experiments on managed search platforms.

### The category filter makes a big difference

Restricting results to the *Landbouw Organisatie en beleid* category improved relevance by 20 to 40 percentage points across configurations. That's a massive lift for a simple filter. It also reveals an important truth: a smarter metadata strategy can partly substitute for a smarter model. Before you fine-tune anything, figure out whether you have useful structure you're not using.

### Tokenization: a hidden culprit?

One hypothesis for the performance differences between models is tokenization. When a model processes text, it first breaks it into tokens—subword units that depend on the tokenizer design. Model 1 uses Byte-Pair Encoding (BPE), Model 2 uses WordPiece (WP), and Model 3 also uses BPE but with a different vocabulary.

For the same Dutch sentence, the number of tokens varies considerably across models. Because Models 1 and 2 have a maximum sequence length of 128 tokens—and 14% to 27% of our document chunks exceed that limit—some content is simply cut off. Model 3's 8192-token limit means it almost never truncates. Whether truncation explains the performance gap is still an open question.

---

## What it cost

Running this on Azure Databricks, the setup cost around EUR 250. Daily operating cost: about EUR 20, mostly due to overcapacity in the vector database. The pilot required roughly 180 person-hours in 2024 (excluding data ingestion work and the work of end users).

For reference, a commercial alternative—a product that ingests 675,000 Dutch policy documents with search, summarization, and chat functionality—costs EUR 39 per month. The "make or buy" question is a legitimate one.

---

## What's next

The pilot proved the concept works: AI search can surface relevant documents that keyword search misses. But there's meaningful work between "useful pilot" and "production tool":

- **Better chunking**: Our current strategy is simple—header, introduction, each policy change separately. More sophisticated chunking could improve both retrieval and context.
- **Fine-tuning on in-domain data**: The *Officiële Bekendmakingen* dataset contains Q&A pairs from the *Tweede Kamer* (Dutch parliament) that could be used to fine-tune a text representation model on exactly the kind of language PBL researchers work with. No one has published such a model on Hugging Face yet—an opportunity to contribute to Dutch NLP.
- **Manual hybrid implementation**: Instead of relying on a black-box managed service, building our own hybrid ranking would let us tune the blend between semantic and keyword signals properly.
- **Evaluation at scale**: One query is a proof of concept, not a benchmark. A real evaluation requires a test set covering the diversity of researcher questions.

---

## Takeaway

If you're building a search system over a specialized document corpus and keyword search isn't cutting it—this kind of pilot is a reasonable first step. Start with AI-only search before you add the complexity of hybrid. Don't assume a bigger model will win. And check your metadata: a well-chosen filter can do more than a model upgrade.

The code and further details are internal to PBL, but the methodology is straightforward to replicate on any similar document corpus with the tools available on Hugging Face and Azure.
