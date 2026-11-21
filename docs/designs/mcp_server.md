# Read-Only EGM MCP Server

*   **Author:** [@Dzuhaib](https://github.com/Dzuhaib)
*   **Status:** Draft
*   **Date:** 2026-11-22
*   **Tracking:** Refs #553 (awaiting `design-ready`)

--------------------------------------------------------------------------------

## Overview

Expose a narrow, read-only subset of EGM to hosts that speak the Model Context
Protocol (MCP), for example ChatGPT, Gemini CLI, or any MCP client. The server
runs over stdio and offers four tools: `list_personas`, `get_persona`,
`get_standard`, and `validate_findings`. It reads only files under the local EGM
source tree and returns text to the host.

This is a new integration surface. Read-only scope reduces risk but does not
remove the need for design review, which this document requests.

## The Problem

MCP-capable hosts cannot invoke EGM skills directly. Users who want a host to
reason with EGM's reviewer personas and standards currently have to copy text by
hand. A small, read-only server lets a host discover personas, read a persona or
standard, and validate the shape of a findings payload without granting write,
shell, or network access.

## Goals

*   Implement a stdio MCP server with a minimal JSON-RPC surface.
*   Expose exactly four read-only tools: `list_personas`, `get_persona`,
    `get_standard`, `validate_findings`.
*   Confine all reads to the EGM source tree and reject path traversal and
    unknown names.
*   Return data only; never execute reviewed content, write files, run shell
    commands, contact the network, or emit telemetry.
*   Cover the server with unit and stdio end-to-end tests.
*   Document the security posture and how to run it.

## Non-Goals

*   Write tools, file creation, or edits.
*   Running reviews, dispatching subagents, or calling models.
*   Network transport, authentication, or multi-user serving.
*   Packaging the server into release artifacts or marketplace listings in this
    iteration.
*   Host-specific adapters or a plugin manifest.
*   Replacing the existing skills or the review loop.

## Open Questions

*   **Question 1**: Should the server be packaged with the plugin or remain a
    source-tree tool? The initial proposal keeps it under `adapters/` and out of
    the release manifest until the interface is reviewed.
*   **Question 2**: What is the minimum MCP protocol version to support? The
    initial proposal implements `initialize`, `tools/list`, and `tools/call`, and
    treats `notifications/initialized` as a no-op.
*   **Question 3**: Should `validate_findings` enforce severity vocabulary and
    required fields, or only JSON shape? The initial proposal validates JSON and
    the presence of findings, and leaves stricter schema for review.

## Critical User Journeys (CUJs)

### CUJ 1: Discover and read a persona

*   **Given**: An MCP host launches the server over stdio.
*   **When**: The host calls `tools/list` and then `get_persona` with a valid
    persona name.
*   **Then**: The server returns the persona list and the canonical persona text.

### CUJ 2: Reject traversal

*   **Given**: A host calls `get_persona` with `../../etc/passwd`.
*   **When**: The server resolves the request.
*   **Then**: The server returns an error and reads nothing outside the EGM
    source tree.

### CUJ 3: Validate findings

*   **Given**: A host supplies a JSON payload with a `findings` array.
*   **When**: The host calls `validate_findings`.
*   **Then**: The server returns `{ "valid": true }`; malformed JSON returns
    `{ "valid": false }` with an error message.

## Technical Plan (Architecture)

The server is a single Python module with no third-party dependencies:

```text
stdio JSON-RPC
     |
     v
dispatch(message)
     |-- initialize        -> serverInfo {name: egm}
     |-- tools/list        -> tool descriptors
     |-- tools/call
     |     |-- list_personas   -> read agents/*.md frontmatter
     |     |-- get_persona     -> validated name -> agents/<name>.md
     |     |-- get_standard    -> validated name -> resources/document_standards/*
     |     |-- validate_findings -> JSON shape check
     |-- notifications/*   -> no response
```

Containment:

*   Persona and standard names are validated against a fixed allowlist derived
    from the source tree; the server rejects separators and `..`.
*   Resolved paths must remain under the repository root.
*   The server never writes, spawns processes, or opens sockets.

## Alternatives Considered

### Alternative A: Expose every skill as an MCP tool

*   **Pros**: More functionality for hosts.
*   **Cons**: Large surface, write and orchestration risk, and unclear semantics
    for long-running review loops.
*   **Reason for Rejection**: Rejected by author: start with the smallest useful
    read-only surface.

### Alternative B: Wrap the CLI in a subprocess

*   **Pros**: Reuses existing entry points.
*   **Cons**: Adds process management and argument-injection risk for little
    benefit; the four tools are simple file reads.
*   **Reason for Rejection**: Rejected by author: in-process reads are simpler and
    easier to confine.

## Non-Functional Requirements

*   **Security & Privacy**: Read-only, local-only, no telemetry, no secrets.
    Traversal and unknown-name requests are rejected.
*   **Performance & Latency**: Each tool call reads one small file; no caching is
    required.
*   **Rollout & Rollback**: The server is additive and not wired into releases in
    this iteration; deleting the adapter and docs reverts it.

--------------------------------------------------------------------------------

### Additional Resources

*   [Model Context Protocol](https://modelcontextprotocol.io/)
*   [Agent personas](../../agents/)
*   [Document standards](../../resources/document_standards/)

--------------------------------------------------------------------------------

## Detailed Implementation Plan

### Target Files to Create/Modify

#### 1. `adapters/egm_mcp/server.py` (Create)

*   **Type / Scope**: Python stdio MCP server.
*   **Proposed Changes**: Implement JSON-RPC dispatch, the four tools, name and
    path containment, and error responses. No third-party dependencies.

#### 2. `docs/mcp.md` (Create)

*   **Type / Scope**: User documentation.
*   **Proposed Changes**: Describe status, tools, how to run the server, and the
    security posture.

#### 3. `tests/unit/test_egm_mcp.py` (Create)

*   **Type / Scope**: Unit and stdio end-to-end tests.
*   **Proposed Changes**: Cover persona discovery, traversal rejection, unknown
    standard rejection, findings validation, unknown tool handling, and an
    initialize/list handshake over stdio.

### Verification Plan

*   **Unit Tests**: `python -m pytest tests/unit/test_egm_mcp.py`
*   **Full Suite**: `python -m pytest tests/unit`
*   **Manual**: Launch `python adapters/egm_mcp/server.py`, send `initialize` and
    `tools/list`, and confirm the response shape.
*   **Security**: Attempt traversal, absolute paths, and unknown names; confirm
    each is rejected without reading outside the tree.

--------------------------------------------------------------------------------

*Drafted with EGM*
