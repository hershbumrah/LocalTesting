# Specification: Test #2

## Summary

Build a backend-first analytics pipeline that scans Git repositories, links commits to Jira ticket IDs, reconstructs per-ticket timelines, computes sprint/reporting metrics, and outputs report-ready data for a future dashboard UI.

This issue appears to be about defining the initial architecture and implementation plan for a system that:
- ingests Git commit history in batch,
- extracts Jira-linked work from commit messages,
- reconstructs effort timelines per story/ticket,
- computes reporting metrics,
- optionally persists historical data,
- exposes data via API and/or report exports for a future frontend.

## Source Material Observations

The attached architecture sketch describes:
- **Frontend (future UI layer)**: React dashboard that displays ticket metrics and summaries.
- **Backend layer**: FastAPI (Python) endpoints for `/run-analysis`, `/metrics`, etc.
- **Core processing engine**: Python-based Git ingestion via GitPython, commit scanning, Jira ID parsing, timeline reconstruction, metrics calculation, and CSV/JSON report generation.
- **Optional data layer**: PostgreSQL for historical tracking and reporting queries.
- **Initial architecture**: Jira Client, Git Client, Core Engine, Report API, Config Client.
- **Planned data handler**: batch ingestion at end of sprint.
- **Questions/unknowns** around dashboard metrics, database necessity, historical comparison, and streaming vs batch ingestion.

## Goals

1. Scan one or more configured Git repositories in batch.
2. Identify commits associated with Jira tickets.
3. Reconstruct timelines for tickets/stories from commit history.
4. Compute metrics per story/ticket and per sprint.
5. Generate consumable report outputs (CSV and JSON at minimum).
6. Expose a backend API for triggering analysis and retrieving results.
7. Keep the architecture extensible for a future React UI and optional historical datastore.

## Non-Goals

- Building the React dashboard UI in this issue.
- Implementing real-time/streaming ingestion unless explicitly added later.
- Implementing full Jira synchronization beyond ticket ID extraction and optional metadata lookup.
- Optimizing with a Rust rewrite in this phase.
- Defining final visual dashboard design.

## Proposed System Architecture

### Core Components

#### 1. Config Client
Responsible for loading runtime configuration:
- repository list
- Jira base URL and credentials, if needed
- analysis options
- output paths
- database connection string, if enabled

#### 2. Git Client / Git Ingestion
Responsible for:
- scanning repository commit history
- reading commit hashes, timestamps, authors, messages, and branches if needed
- providing commits to downstream processing

#### 3. Jira ID Parser
Responsible for extracting ticket IDs from commit messages and/or metadata using configurable patterns such as:
- `ABC-123`
- multiple ticket references in a single commit message
- case-insensitive matching if needed

#### 4. Timeline Engine
Responsible for reconstructing per-ticket timelines from commit sequences:
- first commit referencing a ticket
- last commit referencing a ticket
- duration between milestones
- grouping by sprint or iteration if available
- determining owner/assignee heuristics if supported

#### 5. Metrics Engine
Responsible for computing:
- elapsed time per story/ticket
- session grouping
- iteration count
- tail effort
- per-ticket aggregate metrics
- optional sprint rollups

#### 6. Report API / Exporter
Responsible for:
- producing CSV output
- producing JSON output
- returning report data through backend endpoints
- optionally persisting normalized records to a database

#### 7. Optional PostgreSQL Data Layer
Used for:
- storing computed ticket metrics
- retaining historical snapshots
- supporting comparisons across sprints/releases
- supporting reporting queries without rescanning repositories

## Functional Requirements

### FR1: Repository Scanning
The system must scan one or more configured Git repositories and collect commit data.

#### Required commit fields
- commit hash
- author
- timestamp
- commit message
- repository identifier/path

#### Acceptance criteria
- Given a configured repo, the system can enumerate commits in the selected range.
- The scanner handles repositories with multiple branches if branch selection is configured.
- Scan failures are reported clearly per repository.

### FR2: Jira Ticket Identification
The system must extract Jira ticket IDs from commit messages.

#### Requirements
- Support configurable Jira key patterns.
- Support multiple ticket IDs in one commit message if present.
- Preserve raw commit message for traceability.

#### Acceptance criteria
- A commit message containing a valid Jira issue key is linked to that issue.
- Commits with no Jira key are either ignored or marked unlinked according to configuration.

### FR3: Timeline Reconstruction
The system must reconstruct a timeline per ticket/story from linked commits.

#### Timeline outputs
- first commit timestamp
- last commit timestamp
- number of commits linked to the ticket
- linked authors
- optional inferred work sessions

#### Acceptance criteria
- For each ticket with linked commits, a timeline object is produced.
- Timeline reconstruction is deterministic for the same input dataset.

### FR4: Metrics Computation
The system must compute reportable metrics per ticket and optionally per sprint.

#### At minimum, metrics should include:
- story/ticket ID
- who worked on it
- time spent / elapsed time
- commit count
- first-to-last commit duration
- iteration/session count
- tail effort

#### Acceptance criteria
- Metrics are computed for each ticket with linked commits.
- Metrics are returned in a structured format suitable for export and dashboarding.

### FR5: Report Generation
The system must generate CSV and JSON reports from computed metrics.

#### Output requirements
- CSV export for spreadsheet/report use
- JSON export for API/UI consumption

#### Acceptance criteria
- Reports are produced after a successful analysis run.
- Report files include ticket IDs and computed metrics.

### FR6: Analysis Trigger API
The backend must expose an endpoint to trigger analysis.

#### Suggested endpoints
- `POST /run-analysis`
- `GET /metrics`
- `GET /reports/{report_id}` or similar

#### Acceptance criteria
- Analysis can be started via API.
- The API returns a job/result status.
- Metrics can be retrieved after processing.

### FR7: Optional Historical Storage
The system should optionally persist computed metrics in PostgreSQL.

#### Use cases
- historical sprint comparisons
- report audits
- trend analysis over time

#### Acceptance criteria
- If database persistence is enabled, computed metrics are stored successfully.
- If disabled, the system still functions using file/API outputs.

## Data Model

### Core Entities

#### Repository
- `id`
- `name`
- `path`
- `default_branch`
- `enabled`

#### Commit Record
- `hash`
- `repository_id`
- `author`
- `timestamp`
- `message`
- `linked_ticket_ids`

#### Ticket Timeline
- `ticket_id`
- `repository_id`
- `first_commit_timestamp`
- `last_commit_timestamp`
- `commit_count`
- `authors`
- `linked_commits`

#### Metric Record
- `ticket_id`
- `story_id` [NEEDS CLARIFICATION: if distinct from ticket ID]
- `who_worked_on_it`
- `time_spent`
- `elapsed_time`
- `session_count`
- `iteration_count`
- `tail_effort`
- `sprint_id` [optional]
- `generated_at`

#### Report Snapshot
- `report_id`
- `generated_at`
- `format`
- `location`
- `scope` (repo/sprint/date range)

## API Specification

### POST `/run-analysis`
Triggers a new batch analysis.

#### Request body
- `repository_ids` or repository filters
- optional date range
- optional sprint range
- output format preferences
- persistence toggle

#### Response
- job/run identifier
- status
- summary of queued repositories

### GET `/metrics`
Returns computed metrics for the latest run or a specified run.

#### Query parameters
- `run_id` [optional]
- `repository_id` [optional]
- `ticket_id` [optional]
- `sprint_id` [optional]

#### Response
- array of metric records
- metadata about the run

### GET `/reports/{report_id}`
Returns generated report metadata and/or download location.

#### Response
- report format
- generated timestamp
- file path or download URL
- associated run identifier

## Processing Flow

1. Load configuration.
2. Scan configured Git repositories.
3. Parse commit messages for Jira ticket IDs.
4. Group commits by ticket.
5. Reconstruct ticket timelines.
6. Compute metrics.
7. Generate CSV/JSON outputs.
8. Optionally persist results in PostgreSQL.
9. Return run summary to caller.

## Batch Ingestion Mode

The attached design explicitly indicates **end-of-sprint batch ingestion**.

### Expected behavior
- Analysis runs on demand or on a scheduled batch basis.
- The system processes accumulated commit history, not live event streams.
- Report output is generated after the batch completes.

### Implications
- Simpler implementation than streaming.
- Easier reproducibility and auditability.
- Better fit for sprint-level reporting.

## Implementation Constraints

- Use Python for the core processing engine.
- Use FastAPI for backend endpoints.
- Git history scanning should initially use GitPython or equivalent.
- Report exports should be machine-readable and deterministic.
- Database support should be optional, not required for the first functioning version.

## Open Questions / [NEEDS CLARIFICATION]

1. **Dashboard metrics definition**
   - Which exact metrics must be shown in the future dashboard?
   - Is the minimum set limited to story ID, owner, and time spent, or are additional metrics required?

2. **Story vs ticket terminology**
   - The sketch references both “story” and “Jira ticket.”
   - Are these interchangeable, or should the system distinguish between issue types?

3. **Database requirement**
   - Is PostgreSQL required for MVP, or only optional?
   - Should historical comparison be supported now or later?

4. **Retention requirements**
   - Must old reports be retained for audit purposes?
   - If yes, for how long?

5. **Ingestion mode**
   - Is batch ingestion sufficient for this issue, or should streaming/event-driven ingestion also be designed now?

6. **Jira integration depth**
   - Is extracting ticket IDs from commit messages sufficient?
   - Should the system query Jira for metadata such as summary, assignee, sprint, status, and issue type?

7. **Session/time computation**
   - How should “elapsed time” be computed from commits?
   - What defines a session boundary?
   - What is the exact formula for tail effort?

## Acceptance Criteria

- A configured repository can be scanned successfully.
- Jira-linked commits are identified.
- Ticket timelines are reconstructed.
- Metrics are computed for each ticket.
- CSV and JSON reports are generated.
- The backend exposes an endpoint to trigger analysis.
- The architecture supports optional persistence and future UI integration.

## Suggested Milestone Breakdown

### Milestone 1: Core ingestion and parsing
- implement repo scanning
- implement Jira key extraction
- output raw linked commit records

### Milestone 2: Timeline and metrics
- reconstruct ticket timelines
- compute ticket-level metrics
- produce structured metric objects

### Milestone 3: Reporting API and exports
- implement analysis endpoint
- generate CSV/JSON reports
- implement retrieval endpoints

### Milestone 4: Optional persistence
- add PostgreSQL storage layer
- store historical analysis runs
- support run-based queries

## Risks

- Commit messages may not consistently contain Jira keys.
- Multiple tickets in one commit may complicate attribution.
- “Time spent” inferred from commits may not reflect actual effort.
- Without Jira API integration, sprint/owner metadata may be incomplete.
- Historical comparisons require durable storage and consistent snapshotting.

## Future Enhancements

- Rust refactor for Git ingestion and performance-critical components.
- AI-assisted analytics for summaries and trend interpretation.
- Dashboard views for:
  - ticket metrics
  - sprint summaries
  - team productivity trends
  - adoption reports
- Ad hoc analysis endpoints
- Change/event dashboards and AI adoption trend reporting

## Definition of Done

The issue is complete when:
- the backend can analyze configured repositories in batch,
- Jira-linked commits are parsed and grouped,
- ticket timelines and metrics are generated,
- CSV/JSON outputs are produced,
- and the system is ready for consumption by a future React dashboard or optional database-backed reporting layer.