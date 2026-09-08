---
name: cka-notes
description: Instructions and guidelines for converting raw study notes (*-raw.md) into comprehensive, structured, and exam-focused CKA notes.
---

# CKA Notes Conversion Skill

This skill defines the workflow and quality standards for transforming raw, informal CKA study notes (`<topic>-raw.md`) into structured, comprehensive, exam-ready reference guides (`<topic>.md`).

---

## Core Principles & Guidelines

Every generated note should read as house style, not a template mechanically filled in. In particular: under the **Conceptual Overview**, explain concepts in **clear, simple English without using analogies, metaphors, or real-world comparisons** (e.g., do NOT use analogies like thermostats, airports, control towers, post offices, or shipping ports). Focus directly on how the component actually operates in the system, what concrete problem it solves, and why it works that way (e.g., *"key-value store: stores data without a rigid schema, does not support complex queries, but delivers blazing-fast, flexible key lookups"*). Always pair this direct, simple English explanation with the rigorous, exam-grade Kubernetes definition (e.g., *"etcd is a distributed, consistent, ACID-compliant key-value store implementing the RAFT consensus algorithm that acts as the single source of truth for cluster state"*). Never discard the understanding-level layer in favor of pure jargon, but keep it grounded in technical mechanics rather than external analogies.

### 1. Complete All `@TODO` Items
- Inspect the raw note for any `@TODO` comments or requests (command mapping tables, comparison summaries, diagram requests).
- Thoroughly resolve and expand every `@TODO` with complete, verified technical content.
- Incorporate and embed referenced local assets (e.g. images in `Images/`) using markdown image syntax whenever noted.

### 2. Version & Currency Tagging
- Every topic file must declare the Kubernetes minor version(s) it targets near the top (e.g. *"Verified against v1.31 — flags/behavior may differ on older/newer clusters"*).
- If a command, flag, API group, or component is deprecated, renamed, or recently changed (e.g. dockershim removal, API version bumps), call it out explicitly with a `[!WARNING]` or `[!IMPORTANT]` callout rather than presenting only the current behavior as if it were timeless.
- If the raw note's info might be stale relative to the current exam curriculum, flag it for the user to confirm rather than silently carrying it forward.

### 3. Standardized 9-Part CKA Layout
Every generated `<topic>.md` must follow the consistent layout established across the repository:

1. **Header & Exam Metadata**:
   - Exam Domain (e.g., *Cluster Architecture, Installation & Configuration (25%)*)
   - Weight / Importance
   - Kubernetes version this note targets
   - Allowed Docs Search Keywords (exact search terms permitted on [kubernetes.io/docs](https://kubernetes.io/docs))
2. **Quick-Reference Summary**:
   - A dense, bullet-only summary of the topic's must-know facts, commands, ports, and gotchas — no prose, no diagrams. Meant for last-minute review under time pressure, distinct from the deep-dive sections below.
3. **Conceptual Overview & Mental Model**:
   - High-level summary connecting the concepts.
   - Direct, simple English explanation of the concept from the raw notes (STRICTLY NO analogies or metaphors; explain what it actually does in the system and why) paired with the standard/formal definition.
   - Clean Mermaid flowchart/diagram visualizing architecture, communication paths, or workflows.
4. **Deep-Dive Technical Breakdown**:
   - Component roles, internal mechanics, configuration options, and topology reference tables.
   - Ports, file paths, systemd service vs. static pod designations, config formats.
5. **Command Translation & Mapping Tables**:
   - 1-to-1 imperative comparisons where applicable (e.g., `docker` vs. `nerdctl`, `docker` vs. `crictl`).
   - High-yield flags, subcommands, and outputs.
6. **High-Yield CLI & Imperative Commands**:
   - Practical commands for node/cluster inspection, debugging, and verification.
   - Copy-paste ready snippets with descriptive comments.
7. **Troubleshooting & Diagnostic Runbook**:
   - Decision tree flowchart (Mermaid) for common failure modes.
   - Step-by-step triage sequence (investigate, isolate, resolve, verify).
8. **CKA Exam Tips, Gotchas & Traps**:
   - High-priority callout alerts (`[!WARNING]`, `[!IMPORTANT]`, `[!TIP]`).
   - Exam-specific quirks, edge cases, default timeouts, and common pitfalls.
9. **Self-Test / Active Recall**:
   - A short set of quiz-yourself prompts (5-10) covering the topic's key facts and reasoning, without answers shown inline (or answers collapsed/at the bottom) — meant to be used for recall practice, not re-reading.
   - Include the Official Documentation Bookmarks table here as well: exact page titles, search queries, and documentation anchors allowed during the exam.

### 4. Mermaid Syntax Rules & Error Prevention
To prevent Mermaid rendering and parsing errors across different markdown viewers:
- **Link Syntax**:
  - Solid arrow with text: `A -- text --> B` or `A -->|text| B`
  - Dotted arrow with text: `A -.->|text| B` (**never** write `A -. text .-> B`, which causes a syntax parse error)
  - Dotted line without arrow: `A -. text .- B`
- **Avoid HTML Entities / Bare Ampersands**:
  - Never use a bare `&` in node labels or text; use `and` or `&amp;`.
- **Parentheses & Special Characters**:
  - Always quote node labels containing special characters: `id["Label (Extra Info)"]`
  - Avoid unquoted parentheses inside pipe labels (e.g., use `-->|CRI gRPC|` instead of `-->|CRI (gRPC)|`).
- **Decision Nodes**:
  - Keep question text clean without trailing slashes (e.g., `{"Does socket file exist?"}`).

### 5. Source Traceability
- At the top or bottom of each `<topic>.md`, note which raw file(s) and section(s) it was generated from (e.g. *"Source: `etcd-raw.md`, sections 1-4"*).
- This makes it possible to re-sync the structured note later if the raw note is updated, without re-reading the entire raw file to find what changed.

---

## Transformation Workflow

1. **Read Raw File**: Thoroughly inspect `<topic>-raw.md` for covered topics, intuition/notes, analogies, and `@TODO` tasks.
2. **Extract & Pair Explanations**: Preserve the user's understanding-level explanation written in simple, direct English (strictly avoiding analogies or metaphors) and pair it with the formal Kubernetes architecture definition.
3. **Draft Structured Note**: Populate the 9-part layout, resolving all `@TODO` items and adding relevant diagrams.
4. **Verify Accuracy**: Cross-check technical claims, commands, flags, ports, and file paths against current kubernetes.io/docs (or another authoritative source) rather than trusting the raw note's phrasing as-is. Correct or flag anything that's outdated, imprecise, or version-dependent.
5. **Validate**: Verify Mermaid diagram syntax, relative image links, and command accuracy.
6. **Check for Existing Output**: Before writing, check whether `<topic>.md` already exists. If it does, diff against the new draft or confirm with the user before overwriting rather than silently replacing it.
7. **Output**: Write to `<topic>.md` at the repository root, including the source traceability note.