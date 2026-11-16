# Product

<!-- impeccable:product-schema 1 -->

## Platform

web

## Users

Single-user chat sessions on localhost, used to ask questions, revisit prior answers, and verify whether time-sensitive facts have changed. This is inferred from the existing product limits and the requested chat-agent experience.

## Product Purpose

Since is a chat agent that remembers what it told the user, when it learned it, and whether the source still agrees. Success means a user can ask a follow-up about past advice and receive an honest, evidence-backed answer.

## Positioning

Since stores dated statements and source observations as a diffable record instead of treating a transcript as memory. Follow-ups can recall an earlier baseline, re-check volatile facts, compare observations, and show supporting evidence.

## Operating Context

The primary workflow is a conversational thread with multiple sessions. Users may start a fresh chat, reopen history, load labeled demo history, ask follow-up questions, inspect sources, and expand trace details for a completed turn.

## Capabilities and Constraints

- Preserve streaming chat, sessions, memory recall, source citations, comparison tables, verification states, trace loading, and the existing Fastify/SQLite/LangGraph backend.
- The product is a local, single-user web app with no authentication and no scheduler.
- Demo history is synthetic and must remain visibly labeled.
- The interface must stay responsive and keyboard accessible across desktop, tablet, and mobile.
- File upload, copy, regenerate, search, and export are optional UI enhancements and must not replace or block the existing chat workflow.

## Brand Commitments

The product name is Since. Voice should stay clear, direct, honest about uncertainty, and evidence-led.

## Evidence on Hand

- `DESIGN.md` documents the memory model, evidence rules, architecture, and current product limits.
- Existing UI copy and seeded demo sessions are the source of truth for current user-facing terminology.
- No real user history, testimonials, or production brand assets are available; do not invent them.

## Product Principles

- Remember dates and claims, not just transcripts.
- Separate what was said from what a source showed.
- Report change only when observations are comparable.
- Make uncertainty and provenance visible.
- Keep the chat fast, calm, and easy to resume.

## Accessibility & Inclusion

Maintain keyboard navigation, visible focus, labeled controls, readable contrast, reduced-motion support, touch-friendly targets, and layouts that remain usable at narrow widths and larger text sizes.
