# Project Deal → InterviewAgent: Autonomous Career Negotiation

> Spec date: 2026-06-01
> Source inspiration: [Anthropic Project Deal](https://www.anthropic.com/features/project-deal)
> Target repo for implementation: `hansraj316/InterviewAgent`
> Companion doc: `ARCHITECTURE-AUDIT.md`

---

## Executive Summary

Anthropic's **Project Deal** was a 2025 internal experiment where each participant received a custom Claude agent that autonomously listed, matched, negotiated, and closed deals on their behalf in a shared Slack channel. Pilot outcomes: **186 deals**, 500+ items listed, $4k+ transaction value across 69 Anthropic employees. A measurable but invisible-to-participants gap appeared between model tiers — **Opus agents extracted ~$2.68–$3.64 more per item** than Haiku.

This spec adapts the Project Deal pattern — intake interview → personalized agent → autonomous negotiation loop — into `InterviewAgent`, which already automates the upstream career pipeline (resume tailoring, applications, follow-ups, ~500 raids/day) but stops short of the **offer** stage. We bolt on a `negotiation/` module that handles recruiter conversations under user-controlled autonomy levels.

---

## Why InterviewAgent

Per `ARCHITECTURE-AUDIT.md`, InterviewAgent already implements the pieces Project Deal patterns map onto:

| Project Deal piece | InterviewAgent piece |
|---|---|
| Intake interview to capture preferences | New: `negotiation/intake_interview.py` |
| Personalized Claude system prompt | New: `negotiation/agent.py` built from `NegotiationProfile` |
| Autonomous propose/counter loop | New: `negotiation/loop.py` |
| Shared Slack channel as venue | Reuse `email_notification.py` as primary channel (Slack optional for tests) |
| Private floor + public ask | Floors live in profile, never serialized outbound |
| MCP tool use mid-loop | Reuse `iframe_browser_server.py` for market-data lookup |
| Model-strength matters | Default `claude-opus-4-7`; log model per turn |

This builds on `application_submitter.py`'s agent-with-tools pattern (audit §8 QueryEngine), the existing MCP tool registry (audit §9), and the established MCP transports (audit §17).

---

## Components to Add (in `hansraj316/InterviewAgent`)

### `negotiation/profile.py`
Pydantic schemas:
- `NegotiationProfile` — `target_total_comp`, `base_floor`, `equity_floor`, `signing_floor`, `must_haves` (remote / visa / start_date / title / PTO), `communication_style`, `escalation_triggers`, `walkaway_conditions`
- `OfferTerms` — structured representation of any offer/counter
- `NegotiationState` — per-opportunity state machine (`drafting`, `awaiting_recruiter`, `awaiting_user_approval`, `escalated`, `closed_won`, `closed_lost`, `walked_away`)
- `Turn` — single inbound/outbound exchange + metadata (`model_id`, `policy_mode`, `rationale`)

### `negotiation/intake_interview.py`
Claude-driven intake loop. Elicits the profile, validates ranges (`floor ≤ target`), persists to Supabase. Prompt structure mirrors `application_submitter.py`.

### `negotiation/agent.py`
Composes the per-opportunity system prompt from:
- `NegotiationProfile`
- Opportunity context (company, role, prior thread)
- **BATNA** = other live offers in the InterviewAgent pipeline

Tool surface: `lookup_market_data`, `get_other_offers`, `draft_reply`, `send_reply` (gated), `escalate_to_human`.

Default model `claude-opus-4-7`. `model_id` recorded on every turn so the Project Deal model-strength finding can be reproduced at career-stakes scale.

### `negotiation/loop.py`
Inbound message → state update → draft → policy gate → outbound or queue. State persisted to Supabase between turns; resumable.

### `negotiation/policy.py`
Per-opportunity autonomy modes (default = `draft_only`):

| Mode | Behavior |
|---|---|
| `draft_only` | Draft + rationale shown to user; user approves before send |
| `single_turn` | Auto-send if (a) within profile bounds and (b) confidence ≥ threshold; pause after one turn |
| `full_auto` | Opt-in only; auto-negotiate end-to-end; mandatory escalation on offer < floor, ambiguity, or deadline pressure |

**Hard rule, independent of mode**: final accept/reject *always* requires human approval. Project Deal's no-mid-flight-signoff stance is too risky when stakes are five figures.

### `negotiation/research.py`
Wraps the existing `iframe_browser_server.py` MCP tool for levels.fyi, h1bdata, Glassdoor lookups. Cached per company/role/level.

### Supabase migrations
- `negotiation_profiles` (user_id → profile JSON)
- `negotiations` (opportunity_id, state, autonomy_mode, current_offer)
- `negotiation_turns` (negotiation_id, direction, model_id, content, rationale, policy_applied, sent_at)

Row-level security mandatory before any production use — floors and BATNA data are sensitive.

### Channel adapter
Extend `email_notification.py` with inbound thread classification: detect "this thread is a negotiation" → route to `loop.py`. Optional Slack adapter for sandboxed agent-vs-agent tests (matches Project Deal's original venue).

### `CLAUDE.md` update
Append a Negotiation section so future Claude Code sessions in `InterviewAgent` have working context.

---

## Implementation Phases

| Phase | Scope | Output |
|---|---|---|
| **0 — Meta** *(this repo)* | Spec doc + README callout | This file + link from `README.md` |
| **1 — Profile + Intake** | `profile.py`, `intake_interview.py`, Supabase tables | A user can complete intake and persist a profile |
| **2 — Per-opportunity agent** | `agent.py`, tool wiring, `research.py` | Given an opportunity, the agent can draft one negotiation reply |
| **3 — Loop + autonomy** | `loop.py`, `policy.py`, audit log | Stateful, multi-turn negotiations under all three autonomy modes |
| **4 — Channel adapters** | Email routing in `email_notification.py`; optional Slack | Real recruiter emails flow into the loop |
| **5 — Verification** | Tests + adversarial harness + model A/B | Confidence to enable `full_auto` on a real opportunity |

---

## Verification Plan

- **Unit tests** — state-machine transitions; policy gates reject below-floor sends; final accept/reject always escalates.
- **Replay tests** — canned recruiter threads → snapshot drafts; PRs diff against snapshot.
- **Adversarial harness** — second Claude plays the recruiter; run end-to-end negotiations in sandbox; assert settlement falls within profile bounds.
- **Model A/B** — same scenario set across Opus / Sonnet / Haiku; chart final-TC delta. Goal: reproduce Project Deal's strong-model premium at career stakes.
- **Manual gate** — no real recruiter ever sees a message until per-opportunity autonomy mode is explicitly approved in the dashboard.

---

## Risks (port-specific)

1. **Stakes asymmetry.** Project Deal trades capped at ~$20; a bad salary negotiation costs five figures. Floors must be enforced in code, not just in the prompt.
2. **No-signoff posture is wrong here.** Project Deal explicitly omitted mid-negotiation human approval. We invert that default.
3. **Tone detection / relationship damage.** Recruiters reacting badly to AI-sounding emails can poison a pipeline. Human review of first contact with any new recruiter is mandatory.
4. **Silent model regressions.** Cheap-model usage is invisible until you measure outcomes. Log `model_id` per turn and review monthly.
5. **Privacy.** `NegotiationProfile` and BATNA data are sensitive — Supabase RLS must be in place before any production use.

---

## Open Questions

- Should `full_auto` ever be permitted on a first-round (target-comp ask) message, or only on later rounds where ground truth is bounded by the original ask?
- Recruiter agent-vs-agent: when both sides are agents, do we keep the email-shaped protocol or move to a more structured handshake?
- Where do we surface negotiations in `mission-control-openclaw`? Same pane as InterviewAgent applications, or a dedicated "deals" tab?
