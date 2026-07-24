# Specification: Test #1

## Overview

This issue requests an implementation spike and architecture refactor to support attachment extraction and adapter-based integrations for Forge workflows, with Jira becoming the primary intake surface and GitHub used primarily for codebase execution where needed.

The attached meeting transcript indicates the immediate goals are:

1. Confirm and extract issue attachments such as PDF and image files.
2. Refactor direct GitHub/API interactions into adapter-based services.
3. Separate user-interaction/conversation storage from codebase access.
4. Prepare for Jira-backed workflows, including MMF/user story intake and future bug workflows.
5. Keep “specify” steps aware of the current codebase, while preserving fast conversational interactions where possible.

---

## Goals

### Primary goals
- Detect and extract attachments from incoming issues/items.
- Add a clean adapter layer between the agent/orchestration logic and external systems.
- Support distinct adapters for:
  - user interaction / conversation tracking
  - codebase repository access
  - issue tracker access
- Establish Jira as the intake system for work items.
- Preserve the ability for the system to operate on codebase state during clarify/specify steps where required.

### Secondary goals
- Enable future workflows for bugs and other item types.
- Lay the foundation for integration testing and review-agent cleanup.
- Avoid tightly coupling orchestration logic to GitHub APIs.

---

## Background / Context

The meeting transcript suggests the current implementation has direct integration patterns that should be refactored into adapters. It also suggests a shift from GitHub issues to Jira items as the visible collaborator/intake source, with Jira item types used to trigger different workflows.

Important notes from the transcript:

- Attachments on issues should be extractable and usable as inputs.
- The team expects PDF and image attachment support first.
- There should be a “sharp cleave” between agent logic and GitHub API / storage interactions.
- Codebase checkout is required for certain steps, especially specify/clarify in brownfield contexts.
- Jira should become the main intake surface; GitHub may remain as the execution or codebase surface.
- Some workflows may be automatically triggered based on Jira issue type, such as MMF, user story, or bug.
- The adapter refactor and integration tests are billable to DAS, not the refactor-only spike.

---

## Proposed Scope

### In scope
- Attachment extraction from incoming issue/item payloads.
- Basic support for PDF and image attachments.
- Adapter interface design and refactor of external integrations.
- Separation of:
  - conversation/user interaction
  - codebase repository access
  - issue tracker intake
- Jira intake support and item-type-aware triggering.
- Codebase access during specify/clarify as needed.
- Initial support for a bug workflow as a future extension point.

### Out of scope
- Full Jira migration of all existing processes.
- Full replacement of GitHub as a code execution environment.
- Complete bug-fixing automation end-to-end unless already present.
- Large-scale organizational process changes beyond technical integration.
- Final business rules for MMF/user story routing if not yet agreed.

---

## Functional Requirements

### 1. Attachment extraction
- The system must detect attachments included with an issue/item.
- The system must extract attachment metadata and content when available.
- The system must support at minimum:
  - PDF attachments
  - image attachments
- The system must make attachment data available to downstream workflows such as clarify/specify.
- If attachment extraction is not possible for a given input, the system must fail gracefully and report the missing capability.

### 2. Adapter-based architecture
- External system interactions must be abstracted behind adapters/services.
- The orchestration layer must not directly depend on GitHub APIs or other external APIs.
- At minimum, the architecture should separate:
  - interaction adapter for issue/comment/conversation context
  - codebase adapter for checkout/read operations
  - tracker adapter for Jira intake and work item lookup
- The adapter interfaces should be designed so future integrations can be added without major orchestration changes.

### 3. Codebase access in clarify/specify
- Clarify/specify flows must be able to inspect the current codebase when required.
- The system should support a slower asynchronous path for codebase checkout and inspection.
- The system should preserve a faster conversational path for steps that do not require code access.
- The implementation must make it explicit which steps require codebase access and which do not.

### 4. Jira intake support
- Jira must be supported as the primary intake system for work items.
- The system must be able to detect and process different Jira item types.
- Item type should influence workflow selection, e.g.:
  - MMF
  - user story
  - bug
- The system should support automatic triggering when new eligible Jira items are created.

### 5. Future bug workflow support
- The architecture should allow a workflow where:
  - a bug is opened in Jira
  - the codebase is checked out
  - clarifying questions may be asked
  - the bug is fixed
  - a PR is opened
- The detailed automation for this workflow may remain future work, but the architecture must not block it.

---

## Non-Functional Requirements

- **Modularity:** integrations must be isolated behind interfaces.
- **Extensibility:** new tracker types or execution backends should be easy to add.
- **Reliability:** attachment extraction and code checkout paths must fail gracefully.
- **Performance awareness:** preserve a fast path for non-codebase conversational operations.
- **Maintainability:** refactor should reduce coupling and avoid “spaghetti” direct API calls.
- **Traceability:** adapter boundaries and step requirements should be easy to understand for reviewers.
- **Reviewability:** the code should be structured so review agents can inspect the architecture cleanly.

---

## Architecture Specification

### High-level structure
Introduce a layered design:

1. **Orchestration / Agent Layer**
   - Coordinates workflows
   - Determines step requirements
   - Invokes adapters

2. **Adapter Layer**
   - External system integrations live here
   - Example adapters:
     - Jira adapter
     - issue/conversation adapter
     - codebase adapter
     - attachment adapter/extractor

3. **Domain / Workflow Layer**
   - Business logic for clarify/specify/implement/budget/bug workflows
   - No direct external API calls

### Suggested adapter boundaries
- **Conversation/Issue Adapter**
  - Reads issue text, comments, and attachment references
  - Used for user-facing intake and conversation context

- **Codebase Adapter**
  - Checks out repository state
  - Reads files and source context
  - Supports snapshotting or workspace preparation for agent use

- **Tracker Adapter**
  - Reads Jira items
  - Detects item type and status
  - Resolves item metadata for routing

- **Attachment Extractor**
  - Normalizes attachment references into usable content
  - Handles PDFs and images first

### Execution mode split
- **Fast mode**
  - Pure conversation/metadata processing
  - No codebase checkout

- **Slow mode**
  - Codebase checkout and inspection required
  - Used for specify/clarify when current code state is needed
  - Used for implementation and bug workflows

---

## Data / Interface Requirements

### Attachment model
The system should represent attachments with at least:
- filename
- MIME type or inferred type
- source URL or identifier
- extracted content or binary payload reference
- status of extraction

### Work item model
The system should represent work items with:
- tracker type
- item type
- title
- description/body
- comments
- attachments
- workflow trigger metadata
- repository/codebase target information, if applicable

### Adapter interface expectations
[NEEDS CLARIFICATION: exact language/framework and interface style are not specified in the transcript.]

At a minimum, each adapter should support:
- initialization/configuration
- fetch/read operations
- failure reporting
- test/mocking hooks

---

## Workflow Requirements

### A. Issue/item intake
1. A new Jira item or equivalent trigger is detected.
2. The system reads item type, body, and attachments.
3. Attachment extraction is attempted.
4. The workflow is routed based on item type and required context.

### B. Specify/clarify
1. Determine whether codebase state is needed.
2. If required, use the codebase adapter to inspect the repository.
3. Use attachments and item context to ask clarifying questions.
4. Preserve a quick path if no code access is needed.

### C. Implement / bug workflow
1. Check out codebase.
2. Read relevant files and context.
3. Perform change generation or implementation.
4. Open PR or equivalent output.
5. Track progress via tracker adapter.

---

## Acceptance Criteria

- [ ] Issue attachments can be detected from incoming payloads.
- [ ] PDF and image attachments are extracted or at least normalized into usable inputs.
- [ ] The system no longer couples orchestration directly to GitHub APIs.
- [ ] Adapter interfaces exist for at least conversation/issue, codebase, and tracker concerns.
- [ ] Clarify/specify flows can access current codebase state when needed.
- [ ] A fast conversational path still exists for non-code tasks.
- [ ] Jira items can be ingested and routed by item type.
- [ ] The architecture supports future bug workflows without major redesign.
- [ ] The refactor is structured so integration testing can be added cleanly.
- [ ] The implementation is reviewable and separated into coherent services/adapters.

---

## Testing Requirements

### Unit tests
- Attachment detection and normalization
- Adapter selection / routing logic
- Item type classification and workflow routing
- Failure handling when attachments are missing or unsupported

### Integration tests
- Jira intake payload to workflow start
- Attachment extraction from a Jira issue with PDF/image attachments
- Codebase checkout path for clarify/specify
- Bug workflow routing scaffold

### Manual validation
- Confirm a sample issue with attachments is processed correctly
- Confirm a specify step can access current codebase state
- Confirm a non-code conversational request remains fast
- Confirm Jira item type routing behaves as expected

---

## Rollout / Implementation Notes

- Start with a spike to validate attachment extraction and adapter boundaries.
- Refactor in small steps to avoid breaking existing workflow behavior.
- Keep GitHub codebase access available where needed, but remove direct orchestration coupling.
- Jira work item creation and adapter integration should be treated as the primary platform direction.
- Billable integration work should be limited to actual adapter and integration implementation, not exploratory refactor-only work.

---

## Risks / Considerations

- Attachment extraction may vary significantly by source payload shape.
- Codebase checkout during clarify/specify may increase latency and token/cost usage.
- Jira item-type-driven automation requires clear mapping rules.
- Transitioning from GitHub issues to Jira may require dual support during migration.
- [NEEDS CLARIFICATION: whether GitHub remains the system of record for code-related tasks or becomes only an execution backend.]

---

## Open Questions

1. [NEEDS CLARIFICATION: What is the exact schema for attachments in incoming Jira/GitHub payloads?]
2. [NEEDS CLARIFICATION: Which file types beyond PDF and images should be supported initially?]
3. [NEEDS CLARIFICATION: Should clarify always inspect the codebase, or only when a rule decides it is necessary?]
4. [NEEDS CLARIFICATION: Is Jira the sole intake system, or should GitHub issues remain supported during migration?]
5. [NEEDS CLARIFICATION: What are the exact item-type mappings for MMF, user story, and bug workflows?]
6. [NEEDS CLARIFICATION: Which repository/codebase checkout mechanism should the codebase adapter use?]
7. [NEEDS CLARIFICATION: What should be considered the source of truth for workflow state: Jira, GitHub, or an internal datastore?]
8. [NEEDS CLARIFICATION: What runtime environment and language constraints apply to the adapter refactor?]

---

## Deliverables

- Adapter-based architecture refactor
- Attachment extraction support
- Jira intake support
- Codebase adapter support for clarify/specify
- Basic workflow routing by item type
- Tests for extraction, routing, and adapter boundaries