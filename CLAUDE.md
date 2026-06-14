# CLAUDE.md

## AI Operating Protocol for Obsidian Knowledge Vault

This file defines how Claude should read, create, modify, and organize notes in this Obsidian vault.

The vault is treated as a **structured knowledge graph**, not a simple note repository.

Claude must optimize for:

* conceptual clarity
* connectivity
* atomic knowledge units
* long-term maintainability

---

# 1. Core Philosophy

The vault follows the **Zettelkasten-style knowledge graph model**.

Key principles:

1. Each note represents **one atomic concept**.
2. Notes must be **interconnected via wiki-links**.
3. Knowledge should grow through **linking rather than duplication**.
4. Prefer **many small notes** instead of large documents.
5. The vault should gradually evolve into a **dense semantic graph**.

Claude must prioritize **structure and linking** over prose quality.

---

# 2. Canonical Note Structure

All concept notes should follow this structure:

# Title

## Intuition

A short mental model of the idea.

## Key Ideas

* atomic point
* atomic point
* atomic point

## Explanation

Structured explanation using short paragraphs or bullets.

## Examples

Concrete cases if applicable.

## Related

* [[Related Concept]]
* [[Another Concept]]

## Tags

#concept

Rules:

* One bullet = one idea
* Avoid long paragraphs
* Avoid narrative writing

---

# 3. Linking Rules (Critical)

Obsidian wiki-links are the backbone of the vault.

Claude must apply these rules strictly.

### Always use wiki-links

Use:

[[Concept Name]]

Never use plain text references if a concept exists.

Example:

Correct:
This approach relies on [[Embeddings]] and [[Vector Database]].

Incorrect:
This approach relies on embeddings and vector storage.

---

### Always link important concepts

When a concept appears that could reasonably be its own note, link it.

Example:

Gradient descent updates parameters using the [[Learning Rate]].

---

### Prefer existing note titles

If a concept already exists, reuse its title exactly.

Do not create variants like:

VectorDB
Vector Database System
Vector Storage

Instead reuse:

[[Vector Database]]

---

# 4. Atomicity Rules

Notes must stay small and focused.

Guidelines:

One note = one concept.

If a note contains multiple independent concepts, split them.

Example:

Bad note:

Neural Networks

* backpropagation
* activation functions
* gradient descent

Correct:

[[Neural Network]]
[[Backpropagation]]
[[Activation Function]]
[[Gradient Descent]]

---

# 5. MOC / Index Pages

For larger topics, Claude should create **Map of Content (MOC)** pages.

Structure:

# Topic Name

## Overview

Short explanation.

## Core Concepts

* [[Concept A]]
* [[Concept B]]
* [[Concept C]]

## Subtopics

* [[Subtopic A]]
* [[Subtopic B]]

## Related Areas

* [[Related Field]]

Example:

Machine Learning MOC linking:

* [[Supervised Learning]]
* [[Unsupervised Learning]]
* [[Neural Networks]]
* [[Gradient Descent]]

MOCs should **organize notes without duplicating content**.

---

# 6. Backlink Optimization

When creating a new note, Claude should consider:

Which existing notes should link to it?

Example:

If creating:

[[Vector Database]]

Claude may suggest updates to:

[[Embeddings]]
[[Similarity Search]]
[[RAG]]

to include links to the new note.

Claude should propose backlink improvements when relevant.

---

# 7. Duplicate Concept Prevention

Before creating a new note, Claude should check whether a similar concept already exists.

If a note exists with similar meaning:

Prefer linking instead of creating a new note.

Example:

Existing note:

[[Large Language Model]]

Do not create:

LLM
Large Language Models
Language Model AI

Reuse the canonical title.

---

# 8. Graph Density Goal

The vault should gradually form a dense knowledge graph.

Guidelines:

Each note should ideally have:

3–10 internal links.

Notes with zero links should be avoided.

Claude should suggest links to improve connectivity.

---

# 9. Writing Style Rules

Claude must prefer:

* bullet lists
* short sentences
* conceptual clarity
* minimal verbosity

Avoid:

* essays
* narrative storytelling
* redundant explanations
* large paragraphs

---

# 10. Editing Existing Notes

When modifying notes:

* Preserve existing structure
* Improve clarity
* Add missing links
* Avoid removing meaningful backlinks

Claude should **augment rather than overwrite**.

---

# 11. Handling New Knowledge

When new knowledge is introduced:

1. Identify the main concept
2. Check if it already exists
3. Create an atomic note if needed
4. Link it to related concepts
5. Suggest updates to relevant notes

---

# 12. Output Discipline

When Claude generates notes, it should output:

1. The new or modified markdown note.

2. Optional suggestions for improving links in other notes.

Example suggestion:

Suggested link additions:

[[Embeddings]] → add link to [[Vector Database]]

---

# 13. Forbidden Patterns

Claude must NOT:

* use HTML inside notes
* create duplicate concepts
* write large unstructured essays
* leave obvious concepts unlinked
* mix multiple unrelated ideas in one note

---

# 14. Long-Term Goal

The vault should evolve into a **self-organizing knowledge network**.

Each change should improve:

* conceptual structure
* link connectivity
* knowledge discoverability

Claude acts as a **knowledge architect**, not just a text generator.

