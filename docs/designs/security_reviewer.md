# Security Reviewer Persona

*   **Author:** [@Dzuhaib](https://github.com/Dzuhaib)
*   **Status:** Draft
*   **Date:** 2026-11-22
*   **Tracking:** Refs #<issue> (open a feature-proposal issue and obtain `design-ready` before review)

--------------------------------------------------------------------------------

## Overview

Add a dedicated `security` Goldfish reviewer persona to the EGM review swarm. The
persona reasons adversarially about threat models, trust boundaries, secret
handling, injection surfaces, authorization gaps, and supply-chain risk. It is
wired into the `egm-review-loop` recommendation map, the packaging manifest, and
the OpenCode adapter alongside the existing reviewers.

This design covers only the persona and its wiring. Shared untrusted-content and
hard-prohibition language for every persona is proposed separately in
`prompt_injection_hardening.md`. The read-only MCP surface is proposed separately
in `mcp_server.md`.

## The Problem

EGM reviewers cover architecture (`zoom_out`), comprehension (`comprehension`),
design critique (`design_critic`), implementation readiness
(`implementation_readiness`), codebase details (`codebase_details`), style
(`style`), and inventive contradiction resolution (`inventive_catalyst`). None of
them owns a security lens. As a result, specs that introduce authentication,
authorization, secrets, user-controlled input, file paths, subprocesses, or new
dependencies can pass review without an explicit threat model or mitigation
plan.

Security review is not a substitute for a human security review. It is a
structured first pass that surfaces missing decisions before implementation
locks them in.

## Goals

*   Add a canonical, harness-agnostic `agents/security.md` persona with the same
    read-only and Goldfish-isolation mandates as the other reviewers.
*   Give the persona explicit audit vectors: threat model, secret and credential
    handling, injection and abuse surfaces, dependency and supply-chain risk,
    authorization gaps, data exposure, and operational security.
*   Recommend the persona from `egm-review-loop` for security-relevant specs and
    high-risk features.
*   Package and install the persona across supported runtimes so every host that
    ships the other reviewers also ships `security`.
*   Add behavioral evaluation coverage so the persona's output shape is checked.

## Non-Goals

*   Replacing human security review, threat modeling workshops, or penetration
    testing.
*   Running scanners, contacting networks, executing code, or reading secrets.
*   Changing the existing review stages, severity vocabulary, or report format.
*   Adding the persona to hosts that do not already ship the shared reviewers in
    this iteration.
*   Modifying the shared untrusted-content clauses (tracked by the companion
    design).

## Open Questions

*   **Question 1**: Should `security` be mandatory for every review, or only
    recommended for security-relevant targets? The initial proposal keeps it
    opt-in through the recommendation map to avoid token overhead on unrelated
    docs.
*   **Question 2**: Which existing standard should the persona read when no
    dedicated security standard exists? The initial proposal reuses the
    feature-spec standard and leaves a dedicated standard for a follow-up.
*   **Question 3**: Does the persona need a dedicated `test_scenarios.yaml`, or
    is it covered by the shared review scenarios? The initial proposal adds a
    small co-located suite.

## Critical User Journeys (CUJs)

### CUJ 1: Security-relevant spec review

*   **Given**: A design document that introduces an authenticated endpoint, a
    secret, and a new dependency.
*   **When**: The user runs `/egm-review` and the review loop classifies the
    target as security-relevant.
*   **Then**: The loop recommends the `security` persona, dispatches it with the
    target and standard, and the synthesized report contains threat-model,
    secret-handling, injection, and supply-chain findings with severities and
    mitigations.

### CUJ 2: Persona discovery across runtimes

*   **Given**: A packaged Claude, OpenAI, or OpenCode installation.
*   **When**: The user lists available reviewer personas.
*   **Then**: `security` appears with its canonical `egm-security` name and can be
    dispatched like the other reviewers.

## Technical Plan (Architecture)

The persona follows the existing persona contract:

1.  Frontmatter declares `name: egm-security`, a one-line description, and the
    Apache-2.0 license.
2.  Core mandates repeat the universal Goldfish rules: strict read-only,
    stateless isolation, no chitchat, untrusted content is data, and hard
    prohibitions.
3.  The role section defines adversarial reasoning.
4.  Instructions branch on `Mode: BRAINSTORM` and `Mode: REVIEW`.
5.  Output format emits severity, affected scope, and finding plus recommendation
    in review mode, and threat, risk, and mitigation requirement in brainstorm
    mode.

```text
egm-review-loop
  -> resolve persona path (agents/security.md)
  -> resolve standard (feature-spec standard or closest security standard)
  -> dispatch fresh read-only reviewer
  -> collect findings
  -> synthesizer normalizes and deduplicates
```

Wiring points:

*   `scripts/pack_plugin.py` includes `security` in the required shared agents so
    packaging fails if the persona is missing.
*   `adapters/opencode/install.py` adds `security` to the reviewer list it
    installs and registers.
*   `skills/egm-review-loop/SKILL.md` and its Claude distribution add a
    recommendation for security-relevant specs and high-risk features.

## Alternatives Considered

### Alternative A: Extend `design_critic`

*   **Pros**: No new persona, no extra dispatch, no packaging change.
*   **Cons**: `design_critic` already owns completeness and internal consistency;
    adding security dilutes its rubric and makes findings harder to triage.
*   **Reason for Rejection**: Rejected by author: a dedicated persona keeps the
    security rubric explicit and independently testable.

### Alternative B: Rely on human security review only

*   **Pros**: No token or maintenance cost in EGM.
*   **Cons**: Security decisions still need to be present in the design before a
    human can review them; the swarm currently gives no prompt to surface them.
*   **Reason for Rejection**: Rejected by author: the persona is a structured
    first pass, not a replacement for human review.

## Non-Functional Requirements

*   **Security & Privacy**: The persona is read-only, cannot run shell commands,
    and must not read or reproduce credential material.
*   **Performance & Latency**: One additional reviewer only when recommended;
    token cost is comparable to the existing reviewers.
*   **Rollout & Rollback**: Removing the persona from the recommendation map and
    packaging manifest reverts the behavior without changing existing reviewers.

--------------------------------------------------------------------------------

### Additional Resources

*   [EGM Goldfish Review Protocol & Resolution Engine](review.md)
*   [Agent personas](../../agents/)
*   [Document standards](../../resources/document_standards/)

--------------------------------------------------------------------------------

## Detailed Implementation Plan

### Target Files to Create/Modify

#### 1. `agents/security.md` (Create)

*   **Type / Scope**: Canonical reviewer persona markdown.
*   **Proposed Changes**: Add frontmatter, core mandates, adversarial role,
    brainstorm and review instructions, audit vectors, and output format.

#### 2. `distributions/claude/agents/security.md` (Create)

*   **Type / Scope**: Claude distribution copy.
*   **Proposed Changes**: Mirror the canonical persona in the generated
    distribution.

#### 3. `scripts/pack_plugin.py` (Modify)

*   **Type / Scope**: Packaging manifest.
*   **Proposed Changes**: Add `security` to `REQUIRED_SHARED` agents.

#### 4. `adapters/opencode/install.py` (Modify)

*   **Type / Scope**: OpenCode adapter.
*   **Proposed Changes**: Add `security` to `REVIEWERS`.

#### 5. `skills/egm-review-loop/SKILL.md` and `distributions/claude/skills/egm-review-loop/SKILL.md` (Modify)

*   **Type / Scope**: Review orchestration skill.
*   **Proposed Changes**: Recommend `security` for security-relevant specs and
    high-risk features, with a standard fallback.

#### 6. `skills/egm-review-loop/test_scenarios.yaml` and `TEST.md` (Modify)

*   **Type / Scope**: Behavioral evaluation.
*   **Proposed Changes**: Add a scenario that expects the security recommendation
    and the finding shape.

### Verification Plan

*   **Unit Tests**: `python -m pytest tests/unit`
*   **Packaging**: `python scripts/pack_plugin.py --help` and the packaging tests.
*   **Scenario Coverage**: `python -m pytest tests/unit/test_scenario_coverage.py`
*   **Manual**: Run `/egm-review` on a spec containing an auth boundary and a
    secret, and confirm the security findings and severities.
*   **Static Analysis**: Markdown table and link-integrity tests.

--------------------------------------------------------------------------------

*Drafted with EGM*
