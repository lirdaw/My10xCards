---
project: "10xCards"
version: 1
status: draft
created: 2026-05-22
context_type: greenfield
product_type: web-app
target_scale:
  users: medium
  qps: low
  data_volume: small
timeline_budget:
  mvp_weeks: 3
  hard_deadline: "2026-06-15"
  after_hours_only: true
---

## Vision & Problem Statement

Knowledge workers and professionals who rely on spaced repetition to retain domain knowledge face a consistent friction point: after reading or processing new material, they must manually translate it into flashcards before any retention work can begin. This creation step is time-consuming enough that the habit breaks — either they never start, or their card backlog grows until the system collapses.

The insight: every existing AI-assisted flashcard tool requires too many steps between source text and a usable deck (copy from tool, paste into Anki, format the card, repeat). This product eliminates that gap — paste text, review the AI draft, and the deck is ready. The friction that kills the habit is the workflow, not the willingness to learn.

## User & Persona

**Primary persona**: A knowledge worker or professional in a domain that requires ongoing retention (medical, legal, finance, engineering, or similar). They already believe in spaced repetition as a method — they've tried it before or are trying it now. Their problem is not motivation; it's that the manual cost of flashcard creation consistently outpaces the time they have available after consuming source material.

The moment they reach for this product: they've just read an article, chapter, or document, and they want to convert it into a deck — right now, before the memory fades — but the blank editor is a wall.

## Success Criteria

### Primary
- 75% of AI-generated flashcards are accepted (not rejected) by the user within the review step.
- 75% of all flashcards in the system are created via AI generation, not manual entry.

### Secondary
- Users return for a second review session without external prompting — proves the spaced repetition loop creates habit, not one-shot use.

### Guardrails
- User data (decks and cards) is never lost silently; any persistence failure surfaces as an explicit, user-visible error.

## User Stories

### US-01: User generates a flashcard deck from source text

- **Given** a logged-in user with source material they want to memorize
- **When** they paste the text into the generation input and trigger generation
- **Then** they see a set of AI-generated flashcard drafts they can accept, edit, or reject

#### Acceptance Criteria
- Each draft displays a front (question/term) and back (answer/explanation).
- User can accept a draft as-is (saves to collection), edit it before saving, or reject it (discards).
- After reviewing all drafts, accepted cards appear in the user's collection.
- If generation fails, an explicit error is shown and the user's source text is preserved for retry.

## Functional Requirements

### Authentication
- FR-001: User can register an account with email and password. Priority: must-have
  > Socrates: Counter-argument considered: "registration friction kills first-session conversion." Resolution: standard for a multi-user web product with persistent decks; stands as written.
- FR-002: User can log in and log out with email and password credentials. Priority: must-have
  > Socrates: Counter-argument considered: "password reset is often forgotten, leaving users locked out." Resolution: password reset is a required companion to email+password login — captured as implied scope of this FR.

### Card Generation
- FR-003: User can generate flashcard drafts by pasting source text into the app. Priority: must-have
  > Socrates: Counter-argument considered: "no length guidance means large pastes produce overwhelming output." Resolution: input length guidance / card-count cap is a UX concern captured in NFRs — the FR stands as written.
- FR-004: User can review AI-generated card drafts and accept, edit, or reject each one individually, and can bulk-accept all drafts at once. Priority: must-have
  > Socrates: Counter-argument accepted: "card-by-card review at 20+ cards is more work than it saves — bulk accept/reject is missing." Resolution: FR expanded to include bulk-accept shortcut alongside individual review.

### Card Management
- FR-005: User can create a flashcard manually without AI assistance. Priority: must-have
  > Socrates: Counter-argument considered: "manual creation is an edge case (< 25% of cards) that duplicates the edit flow." Resolution: manual creation is also the fallback path when AI generation fails (guardrail); stands as must-have.
- FR-006: User can view all flashcards in their collection. Priority: must-have
  > Socrates: Counter-argument considered: "'view all' without search/filter becomes unusable at 100+ cards." Resolution: known tech-debt risk, accepted for MVP; search/filter is a v2 addition.
- FR-007: User can edit a saved flashcard. Priority: must-have
  > Socrates: Counter-argument considered: "if the accept/edit flow is good, post-save edits are rare." Resolution: users will discover errors after save; edit is necessary for quality control; stands as written.
- FR-008: User can delete a saved flashcard. Priority: must-have
  > Socrates: Counter-argument considered: "permanent deletion without undo is unrecoverable and erodes trust." Resolution: a confirmation step (not soft delete) is sufficient mitigation for MVP; stands as written.

### Review Session
- FR-009: User can start a spaced repetition review session from their card collection. Priority: must-have
  > Socrates: Counter-argument considered: "on day 1 the SRS algorithm may schedule no cards as due, breaking the first-session loop." Resolution: a UX affordance (e.g., immediate review mode for new cards) must be considered during implementation; FR stands, UX constraint noted.
- FR-010: User can rate each card during a review session to drive the spaced repetition algorithm. Priority: must-have
  > Socrates: Counter-argument considered: "rating granularity is coupled to the algorithm — choosing wrong means reworking both." Resolution: rating interface is a downstream design concern coupled to the SRS library chosen; FR stands, algorithm selection is flagged for stack selection step.

## Non-Functional Requirements

- AI generation provides continuous visible progress feedback during any operation that takes longer than two to three seconds; the user is never left with a blank screen while generation runs.
- Each user's flashcard collection is strictly isolated — no card belonging to user A is ever accessible to user B, under any condition.
- If AI generation fails, the user's source text is preserved and available to retry without re-pasting.
- The product is fully usable on the latest two major versions of the four mainstream desktop browsers (Chrome, Firefox, Safari, Edge).

## Business Logic

The app converts raw source text into study-ready flashcards by deciding what's worth remembering and how to phrase it as a question.

The user supplies any pasted text — no format constraint, no length limit — as the single input. The product examines it and produces a set of flashcard drafts, each consisting of a front (question or prompt) and back (answer or explanation), chosen and phrased to represent the key ideas in the source. The user then applies the final judgment — accepting, editing, or rejecting each draft — before any card is committed to their collection.

Spaced repetition scheduling — when each card surfaces for review — is outside this product's domain rule. The product owns the generation quality decision; scheduling is delegated to a ready-made implementation and is not a rule this product defines.

## Access Control

Multi-user web app. Access via email + password login.

- Sign-up: email + password (standard registration flow). Password reset is in scope.
- Sign-in: email + password credentials.
- User model: flat — all authenticated users have identical capabilities (generate, create, edit, review, delete flashcards).
- No admin role in MVP.
- Unauthenticated users reach a login/signup wall; no public content access.

## Non-Goals

- **No custom spaced repetition algorithm**: the product uses an off-the-shelf spaced repetition library. Building a proprietary algorithm is explicitly out of scope for MVP and beyond.
- **No multi-format import**: text paste is the only source input. PDF, DOCX, URL fetch, and file upload are v2+ features.
- **No flashcard sharing between users**: all decks and cards are private to their owner. No public decks, collaboration, or sharing links.
- **No mobile app**: native iOS or Android is out of scope. A responsive web experience is acceptable; a native app is not planned.
- **No integrations with external educational platforms**: no export to Anki, no LMS integrations, no third-party learning platform connectors.

## Open Questions

1. **Which spaced repetition algorithm and rating model will this product use?** — The number of rating buttons and the scheduling behavior are coupled to the algorithm choice. Owner: tech-stack-selector step. Blocking: FR-010 UI design until resolved.
2. **First-session UX when no cards are immediately scheduled for review** — On day 1, the scheduling algorithm may not immediately surface any due cards. A "review all new cards now" mode or equivalent affordance is needed to prevent an empty review session on first use. Owner: implementation design. By: before FR-009 implementation begins.
