# Plan: Procode Lossless Transcript Refiner

## TL;DR

Replace the declarative M365 Copilot agent with a **procode solution**: a **React SPA** + **Azure Functions API** (TypeScript) + **Azure Storage** (Blob/Queue/Table) for async job processing, with **Azure OpenAI** for LLM calls. The agent programmatically parses transcripts, chunks them into speaker-turn batches, processes them through a multi-step LLM pipeline, and generates output files deterministically. Includes a **clarification Q&A flow** — users review flagged issues, submit clarifications, and the pipeline re-runs. A **Copilot API Plugin** exposes the same API to M365 Copilot. Architecture follows **interfaces + DI**, **adapter pattern** for all integrations, and **TDD** throughout. **Anonymous access** for V1 — no auth on SPA or API. Function App **managed identity** for Azure resource access.

---

## Architecture

```
                     ┌─────────────────────┐
                     │  M365 Copilot        │
                     │  (API Plugin via      │
                     │   OpenAPI spec)       │
                     └─────────┬────────────┘
                               │
┌──────────────┐    ┌──────────▼────────────┐    ┌─────────────────────┐
│  React SPA   │───►│  Azure Functions API  │───►│  Azure Storage      │
│  (Static     │    │  (HTTP triggers)      │    │  ├─ Blob (transcripts│
│   Web Apps)  │    │  POST /api/jobs       │    │  │   + outputs)      │
│              │◄───│  PUT  /api/jobs/:id/* │    │  ├─ Queue (job queue)│
│  polls       │    │  GET  /api/jobs/:id   │    │  └─ Table (job state)│
│  /status     │    └───────────────────────┘    └──────────┬──────────┘
└──────────────┘         │ managed identity                 │ queue trigger
                         ▼                    ┌─────────────┘
                     ┌──────────────┐         │
                     │  Azure OpenAI│    ┌────▼─────────────────┐
                     │  (GPT-4o)    │◄───│  Azure Function      │
                     └──────────────┘    │  (Queue trigger)     │
                                         │  processJob          │
                                         │  ├─ Parse transcript  │
                                         │  ├─ Chunk into batches│
                                         │  ├─ LLM pipeline     │
                                         │  ├─ Generate files    │
                                         │  └─ Write outputs     │
                                         └──────────────────────┘
```

**Auth model**: Anonymous access to SPA and API. Function App uses **managed identity** to authenticate to Azure Storage and Azure OpenAI. No user auth, tokens, or MSAL in V1.

---

## API Surface

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/api/jobs` | Create job → returns `{ jobId, uploadUrl }` (SAS) |
| POST | `/api/jobs/inline` | Create + submit inline → accepts transcript text in body (for Copilot) |
| PUT | `/api/jobs/:id/submit` | Verify blob uploaded, enqueue for processing → 202 |
| GET | `/api/jobs/:id` | Job status + progress + output SAS URLs when complete |
| PUT | `/api/jobs/:id/clarify` | Submit clarification responses → re-enqueue for re-processing → 202 |

### Job State Machine

```
created → uploaded → queued → parsing → compressing → verifying → clarifying → digesting → generating
  → completed (with clarificationNotes)
    → awaiting_clarification (user reviews)
      → requeued → parsing → ... → completed (final)
  → failed (at any point)
```

---

## Project Structure

```
LosslessTranscriptRefiner/
├── frontend/                          # React SPA
│   ├── src/
│   │   ├── components/
│   │   │   ├── UploadForm.tsx         # File picker, create → upload → submit
│   │   │   ├── JobProgress.tsx        # Polls status, shows pipeline stage
│   │   │   ├── OutputDownloads.tsx    # Download links for completed outputs
│   │   │   └── ClarificationReview.tsx # Displays issues, response form, skip/submit
│   │   ├── services/
│   │   │   └── apiClient.ts           # Backend API calls
│   │   └── App.tsx
│   └── package.json
│
├── api/                               # Azure Functions (TypeScript, v4 model)
│   ├── src/
│   │   ├── functions/                 # Thin function entry points (wire DI, delegate)
│   │   │   ├── createJob.ts           # POST /api/jobs
│   │   │   ├── createJobInline.ts     # POST /api/jobs/inline
│   │   │   ├── submitJob.ts           # PUT /api/jobs/:id/submit
│   │   │   ├── getJobStatus.ts        # GET /api/jobs/:id
│   │   │   ├── submitClarifications.ts # PUT /api/jobs/:id/clarify
│   │   │   └── processJob.ts          # Queue trigger
│   │   │
│   │   ├── interfaces/                # All contracts
│   │   │   ├── IBlobStorage.ts
│   │   │   ├── IQueueService.ts
│   │   │   ├── ITableStorage.ts
│   │   │   ├── ILlmClient.ts
│   │   │   ├── ITranscriptParser.ts
│   │   │   ├── IChunker.ts
│   │   │   ├── IPipelineStep.ts
│   │   │   ├── IPipelineOrchestrator.ts
│   │   │   ├── IFileGenerator.ts
│   │   │   └── IJobService.ts
│   │   │
│   │   ├── adapters/                  # Thin SDK wrappers — NO logic, NOT unit-tested
│   │   │   ├── azureBlobAdapter.ts    # Wraps @azure/storage-blob, uses DefaultAzureCredential
│   │   │   ├── azureQueueAdapter.ts   # Wraps @azure/storage-queue, uses DefaultAzureCredential
│   │   │   ├── azureTableAdapter.ts   # Wraps @azure/data-tables, uses DefaultAzureCredential
│   │   │   └── azureOpenAiAdapter.ts  # Wraps @azure/openai, uses DefaultAzureCredential
│   │   │
│   │   ├── services/                  # Business logic — fully tested
│   │   │   ├── transcriptParser.ts    # Deterministic regex parser for Teams format
│   │   │   ├── chunker.ts            # Token-safe speaker-turn batching
│   │   │   ├── jobService.ts         # Job lifecycle (create, submit, status, clarify, update)
│   │   │   └── fileGenerator.ts      # Programmatic .txt / .docx creation
│   │   │
│   │   ├── pipeline/                  # LLM pipeline steps — fully tested
│   │   │   ├── conciseTranscript.ts   # Step A: chunk-by-chunk compression
│   │   │   ├── completenessCheck.ts   # Step B: verification pass
│   │   │   ├── clarificationNotes.ts  # Step C: transcription issue detection
│   │   │   ├── structuredDigest.ts    # Step D: topic reorganisation
│   │   │   └── orchestrator.ts        # Coordinates A→B→C→D, status updates
│   │   │
│   │   ├── prompts/                   # System prompts extracted from Instructions.md
│   │   │   ├── compression.ts         # Accepts optional clarifications to inject
│   │   │   ├── completeness.ts
│   │   │   ├── clarification.ts
│   │   │   └── digest.ts
│   │   │
│   │   ├── models/                    # Shared types
│   │   │   ├── speakerTurn.ts
│   │   │   ├── transcriptChunk.ts
│   │   │   ├── jobStatus.ts
│   │   │   ├── clarificationIssue.ts
│   │   │   └── pipelineResult.ts
│   │   │
│   │   └── config/
│   │       └── container.ts           # Manual constructor injection wiring
│   │
│   ├── tests/
│   │   ├── services/                  # Unit tests for all services
│   │   ├── pipeline/                  # Unit tests for all pipeline steps
│   │   ├── functions/                 # Integration tests for HTTP/queue functions
│   │   └── fixtures/                  # Sample transcripts for testing
│   │
│   ├── host.json
│   ├── local.settings.json
│   ├── package.json
│   └── tsconfig.json
│
├── copilot-plugin/                    # M365 Copilot API Plugin
│   ├── appPackage/
│   │   ├── manifest.json
│   │   ├── ai-plugin.json
│   │   └── openapi.yaml
│   └── instructions.md
│
└── infra/                             # IaC
    └── main.bicep                     # Storage, Function App (managed identity), Static Web App, AOAI
```

---

## Implementation Phases

### Phase 1: Foundation — Project Setup & Interfaces (Steps 1–4)

1. Scaffold Azure Functions v4 TypeScript project in `api/`
   - `npm init`, configure `tsconfig.json`, `host.json`, `local.settings.json`
   - Dev deps: `vitest`, `@azure/functions` v4
   - Runtime deps: `@azure/storage-blob`, `@azure/storage-queue`, `@azure/data-tables`, `@azure/openai`, `@azure/identity`, `docx`, `uuid`

2. Scaffold React SPA in `frontend/`
   - Vite + React + TypeScript
   - No auth libraries needed

3. Define ALL interfaces in `api/src/interfaces/`
   - `IBlobStorage`: `generateUploadSas(jobId)`, `generateDownloadSas(blobPath)`, `exists(blobPath)`, `readBlob(blobPath)`, `writeBlob(blobPath, content)`
   - `IQueueService`: `enqueue(queueName, message)`
   - `ITableStorage`: `createEntity(table, entity)`, `getEntity(table, partitionKey, rowKey)`, `updateEntity(table, entity)`
   - `ILlmClient`: `complete(systemPrompt, userMessage, options?)` → `string`
   - `ITranscriptParser`: `parse(rawText)` → `SpeakerTurn[]`
   - `IChunker`: `chunk(turns, maxTokens)` → `TranscriptChunk[]`
   - `IPipelineStep`: `execute(input, context)` → `StepResult`
   - `IPipelineOrchestrator`: `run(jobId, rawTranscript, options?)` → `PipelineResult` — options includes optional `clarifications: ClarificationResponse[]`
   - `IFileGenerator`: `generateTxt(content)`, `generateDocx(content)` → `Buffer`
   - `IJobService`: `create()`, `submit(jobId)`, `getStatus(jobId)`, `updateStatus(jobId, status, outputUrls?)`, `submitClarifications(jobId, responses)`

4. Define shared types/models in `api/src/models/`
   - `SpeakerTurn`: `{ speaker, timestamp, content, index }`
   - `TranscriptChunk`: `{ turns, overlapTurns, chunkIndex, tokenCount }`
   - `JobStatus`: `{ jobId, status, progress?, outputUrls?, clarificationNotes?, error?, createdAt, updatedAt }`
   - `ClarificationIssue`: `{ issueId, title, relevantText, speaker, timestamp, issue, reason, clarificationNeeded }`
   - `ClarificationResponse`: `{ issueId, response: string | 'skip' }`
   - `PipelineResult`: `{ conciseTranscript, clarificationNotes, structuredDigest }`

### Phase 2: Core Services — TDD (Steps 5–8)

*All steps: tests written first.*

5. **`transcriptParser.ts`** — implements `ITranscriptParser`
   - Regex patterns for Teams copy-paste format: `<speaker name>\n<timestamp>\n<content lines>`
   - Handle: multi-line content, empty lines, Unicode speakers
   - Tests: valid transcripts, malformed input, edge cases, empty input
   - *Depends on: step 3*

6. **`chunker.ts`** — implements `IChunker`
   - Groups `SpeakerTurn[]` into ~2,000–3,000 token batches
   - Token estimation: words × 1.3
   - Never splits a turn; 2–3 overlap turns for context
   - Tests: token limits, no split turns, overlap, single-turn, sub-chunk transcripts
   - *Depends on: step 3*

7. **`jobService.ts`** — implements `IJobService`
   - Injects `ITableStorage`, `IBlobStorage`, `IQueueService`
   - `create()`: GUID, Table entity (status: "created"), SAS → `{ jobId, uploadUrl }`
   - `submit(jobId)`: verify blob, update Table ("queued"), enqueue `{ jobId }`
   - `getStatus(jobId)`: read Table; if "completed", generate output SAS URLs; include clarificationNotes if present
   - `updateStatus(jobId, status, data?)`: update Table
   - `submitClarifications(jobId, responses)`: store responses in Table/Blob, update status to "requeued", re-enqueue `{ jobId, clarifications: true }`
   - Tests: all flows with mocked adapters, error cases
   - *Depends on: step 3*

8. **`fileGenerator.ts`** — implements `IFileGenerator`
   - `.txt`: plain text formatting
   - `.docx`: `docx` npm package — headings, speaker formatting
   - Tests: format correctness, content integrity
   - *Depends on: step 3*

### Phase 3: LLM Pipeline — TDD (Steps 9–14)

*Pipeline steps tested with mocked `ILlmClient`.*

9. **System prompts** (`api/src/prompts/`)
   - `compression.ts`: substantive/non-substantive rules, priority order, output format. **Accepts optional `ClarificationResponse[]`** — when present, injects "The user has provided the following clarifications: ..." into the system prompt so the LLM applies them during compression.
   - `completeness.ts`: retention checklist
   - `clarification.ts`: issue detection rules, structured output format
   - `digest.ts`: topic reorganisation, full-fidelity requirement
   - Each exports a prompt-builder function
   - *Depends on: nothing (pure functions)*

10. **`conciseTranscript.ts`** — Step A: Compression
    - Per chunk: system prompt (with optional clarifications) + speaker turns → `ILlmClient.complete()`
    - Parse response back to `SpeakerTurn[]`
    - Reassemble chunks, deduplicate overlap turns
    - Sequential processing
    - Tests: prompt construction (with/without clarifications), response parsing, overlap dedup, error handling
    - *Depends on: 3, 9*

11. **`completenessCheck.ts`** — Step B: Verification
    - Source vs. concise comparison via LLM
    - Re-compress specific chunks if issues found (max 1 retry)
    - Tests: identifies missing content, triggers re-compression, respects retry cap
    - *Depends on: 3, 9, 10*

12. **`clarificationNotes.ts`** — Step C: Issue Detection
    - Send concise transcript with clarification prompt
    - Parse structured `ClarificationIssue[]` from LLM response
    - Tests: issue parsing, "no issues" case, chunked input
    - *Depends on: 3, 9*

13. **`structuredDigest.ts`** — Step D: Topic Reorganisation
    - Two-pass: identify topics, reorganise content
    - Tests: all points present, proper headings
    - *Depends on: 3, 9*

14. **`orchestrator.ts`** — implements `IPipelineOrchestrator`
    - `run(jobId, rawTranscript, options?)`:
      - If `options.clarifications` provided: pass to compression prompt, skip Step C (clarification notes already resolved)
      - Otherwise: full pipeline parse → chunk → A → B → C → D → generate files
    - Updates job status at each stage via `IJobService`
    - Per-chunk error recovery
    - Tests: full flow with mocks, re-run with clarifications, partial failure, status updates
    - *Depends on: 5, 6, 8, 10–13*

### Phase 4: Adapters & Azure Function Wiring (Steps 15–18)

15. **Adapters** — NO logic, NOT tested
    - All use `DefaultAzureCredential` from `@azure/identity` (resolves managed identity in Azure, local dev via `az login`)
    - `azureBlobAdapter.ts`, `azureQueueAdapter.ts`, `azureTableAdapter.ts`, `azureOpenAiAdapter.ts`
    - *Depends on: step 3*

16. **`container.ts`** — manual DI wiring
    - Reads env vars: `STORAGE_ACCOUNT_NAME`, `AOAI_ENDPOINT`, `AOAI_DEPLOYMENT`
    - No connection strings or API keys — managed identity handles auth
    - `createContainer()` → fully wired object graph
    - *Depends on: 5–8, 10–15*

17. **Function entry points**
    - `createJob.ts`: POST → `jobService.create()` → `{ jobId, uploadUrl }`
    - `createJobInline.ts`: POST → write transcript to blob, `jobService.submit()` → `{ jobId }`
    - `submitJob.ts`: PUT → `jobService.submit(jobId)` → 202
    - `getJobStatus.ts`: GET → `jobService.getStatus(jobId)` → `JobStatus` (includes `clarificationNotes` when status is "completed")
    - `submitClarifications.ts`: PUT → validate responses against stored issues, `jobService.submitClarifications(jobId, responses)` → 202
    - `processJob.ts`: Queue → read message, check for clarifications flag, call `orchestrator.run(jobId, transcript, { clarifications })` → update status
    - Input validation on all HTTP endpoints (GUID format, existence checks)
    - *Depends on: step 16*

18. **Integration tests** for function entry points
    - HTTP functions with real DI but mocked adapters
    - Queue function with mocked adapters
    - Clarification submission + re-processing flow
    - *Depends on: step 17*

### Phase 5: Frontend (Steps 19–23) *parallel with Phase 4*

19. **API client** (`frontend/src/services/apiClient.ts`)
    - `createJob()` → POST /api/jobs
    - `uploadTranscript(uploadUrl, file)` → PUT to Blob SAS URL
    - `submitJob(jobId)` → PUT /api/jobs/:id/submit
    - `getStatus(jobId)` → GET /api/jobs/:id
    - `submitClarifications(jobId, responses)` → PUT /api/jobs/:id/clarify
    - `downloadOutput(url)` → GET output SAS URL

20. **UploadForm component** — file picker (.txt), triggers create → upload → submit, navigates to status view

21. **JobProgress component** — polls `/api/jobs/:id`, shows current pipeline stage with visual progress

22. **OutputDownloads component** — renders download links for .txt + .docx of each output (concise transcript, clarification notes, structured digest)

23. **ClarificationReview component** — renders when status is "completed" and clarificationNotes has issues:
    - Displays each `ClarificationIssue` as a card: relevant text, issue description, reason
    - Each card has a text input for user's response + "Skip" button
    - "Submit All" button → calls `submitClarifications()` → navigates back to JobProgress (which now polls the re-run)
    - "Accept as-is" button → no re-processing, user downloads outputs directly

### Phase 6: Copilot API Plugin (Steps 24–25) *depends on Phase 4*

24. **`openapi.yaml`** — describes all endpoints including `/api/jobs/:id/clarify`
    - `POST /api/jobs/inline` for Copilot (transcript text in body)
    - Copilot flow: create inline → poll status → present clarification notes to user → submit clarifications → poll → return output URLs

25. **`instructions.md`** — Copilot orchestration instructions
    - Clarification Q&A: when status is "completed" with clarification notes, present each issue to the user one at a time, collect responses, submit via clarify endpoint

### Phase 7: Infrastructure & Deployment (Steps 26–28) *depends on all*

26. **`infra/main.bicep`**
    - Azure Storage Account (Blob: `transcripts`, `outputs`; Queue: `jobs`; Table: `jobs`)
    - Azure Function App (Node.js 20 LTS, consumption plan, **system-assigned managed identity**)
    - Role assignments: Storage Blob Data Contributor, Storage Queue Data Contributor, Storage Table Data Contributor on Storage Account → Function App identity
    - Role assignment: Cognitive Services OpenAI User on AOAI resource → Function App identity
    - Azure Static Web App (React SPA) — anonymous
    - Azure OpenAI resource (GPT-4o deployment)
    - App settings: `STORAGE_ACCOUNT_NAME`, `AOAI_ENDPOINT`, `AOAI_DEPLOYMENT`

27. **Deploy and test end-to-end**

28. **Copilot plugin registration** in M365 admin center

---

## Verification

1. **Unit tests (TDD)**: All services and pipeline steps tested with mocked interfaces via `vitest`
2. **Integration tests**: Function entry points with real DI but mocked adapters — includes clarification flow
3. **Parser tests**: Real Teams transcript samples — multiple speakers, timestamps, edge cases
4. **Pipeline tests**: Small 5-turn transcript → all 3 outputs correct format/content
5. **Pipeline re-run test**: Submit clarifications → verify prompts include clarifications → verify updated outputs
6. **Scale test**: 2+ hour transcript (~30K words) → no lost chunks, complete outputs
7. **File generation test**: `.txt` and `.docx` well-formed
8. **Clarification flow E2E**: Upload → complete → review clarifications → submit → re-run → download final outputs
9. **Copilot plugin test**: Invoke via M365 Copilot → full lifecycle including clarification Q&A

---

## Decisions

| ID | Decision |
|----|----------|
| D1 | **Azure OpenAI** for LLM. Async queue has no user context for delegated Graph calls. `ILlmClient` interface allows future swap. |
| D2 | **Blob + Queue + Table** async architecture. SAS upload for large files. |
| D3 | **Azure Function** queue trigger for processing. Isolated, auto-scales. |
| D4 | **React SPA** on Static Web Apps. Polls /status. |
| D5 | **API Plugin** for Copilot integration (OpenAPI spec). Same endpoints. |
| D6 | **Manual constructor injection**. No DI framework. Single `container.ts`. |
| D7 | **Adapters**: pure pass-through SDK wrappers. No logic. Not unit-tested. |
| D8 | **TDD**: tests written first for all services and pipeline steps. |
| D9 | **Anonymous** SPA/API for V1. **Managed identity** for all Azure resource access. No connection strings or API keys. |
| D10 | **Clarification Q&A in V1**. PUT /api/jobs/:id/clarify endpoint, `ClarificationReview` frontend component. On re-run, clarifications injected into compression prompt; Step C skipped. |
| D11 | **Poison messages**: `processJob` catches non-transient errors → "failed" status, does NOT throw. Throws on transient errors for queue retry. |
| D12 | **Inline endpoint** for Copilot: POST /api/jobs/inline (text in body, no SAS dance). |
| D13 | **Teams copy-paste** transcript format (primary). VTT as future extension. |

---

## Technical Principles

- **Interfaces everywhere**: Every integration point and service has an interface. All dependencies are injected.
- **Adapter pattern**: All SDK/external integrations are wrapped in thin adapters that contain NO logic — they are pure pass-through to the SDK. Adapters implement the corresponding interface. Adapters are NOT unit-tested (they are tested via integration tests only).
- **Manual DI**: A single `container.ts` file wires all dependencies using constructor injection. No DI framework.
- **TDD**: Tests are written before implementation for all services and pipeline steps. Use `vitest`. Adapters are excluded from unit testing.
- **Managed identity**: All Azure SDK adapters use `DefaultAzureCredential`. In Azure this resolves to managed identity; in local dev it resolves to `az login` credentials.
- **No secrets in config**: Environment variables contain only resource names/endpoints (e.g., `STORAGE_ACCOUNT_NAME`), never connection strings or API keys.

---

## Appendix A: Original Business Rules (from Instructions.md)

The following rules from the original declarative agent must be preserved in the system prompts. These are the source of truth for what the LLM should do.

### Purpose

Transform a meeting transcript into three outputs without losing substantive meaning:
1. Concise Transcript
2. Clarification Notes
3. Structured Transcript Digest

This is not summarisation. It is lossless semantic compression and reorganisation of substantive content.

### Priority (apply in order)

1. Preserve substantive meaning.
2. Exclude non-substantive conversational content.
3. Preserve ambiguity, uncertainty, disagreement, and incompleteness.
4. If unsure whether content is substantive, prefer retaining it in the Concise Transcript.
5. Prefer fidelity over elegance.

### Core Rules

- Use only the information contained in the provided transcript.
- Do not add, infer, assume, interpret, or fill gaps from prior knowledge or external knowledge.
- Do not convert tentative statements into facts.
- Do not omit substantive information.
- Do not introduce recommendations, conclusions, or analysis that are not explicitly supported by the transcript.
- Preserve ambiguity where ambiguity exists in the source.
- Preserve speaker attribution for retained content.
- Exclude non-substantive meeting chatter by default.

### Substantive Information

Includes any: fact, request, question, answer, decision, action item, requirement, risk, issue, assumption, dependency, commitment, concern, objection, correction, uncertainty, unresolved item.

### Non-Substantive Content

Exclude unless it materially affects interpretation:
- greetings and farewells
- pleasantries such as "nice to meet you"
- thanks or courtesy with no business meaning
- audio/video checks
- attendance checks
- meeting setup or logistics
- filler, false starts, and repeated phrasing adding no meaning
- social small talk
- transitional phrases with no substantive content

Default: when in doubt, exclude conversational or social content unless it carries business, technical, or decision-making significance.

### Output 1: Concise Transcript

Requirements:
- Keep transcript format and retained speaker order
- Preserve speaker attribution
- Remove non-substantive content
- Rewrite retained speaker turns concisely while preserving meaning
- Omit turns with no substantive information
- Keep substantive conversational sequencing (e.g., answers after questions)
- Compress wording, conjunctions, and sentence structure
- Remove hedging, qualifiers, discourse markers, filler — unless substantive
- Do NOT remove repetition where it shows emphasis, concern, or disagreement

Format per speaker turn:
```
<speaker> (<timestamp>)
<concise transcription>
```

### Output 1 Verification: Completeness Check

Compare concise transcript to source. Verify retention of: decisions, actions, dates, numbers, commitments, risks, concerns, requirements, open questions, dependencies, unresolved items. Verify non-substantive content excluded.

### Output 2: Clarification Notes

Check for: conflicting statements, inconsistent numbers/dates/names/terminology, nonsensical/garbled/incomplete statements, likely speaker attribution errors.

Do NOT: ask for additional background/context, suggest information not discussed.

Rules: flag but do not resolve, do not silently correct, do not infer from external knowledge, only raise plausible material issues.

Format per issue:
```
<issue ID> - <issue title>
  - Relevant Text (<speaker>, <timestamp>)
    <relevant text>
  - Issue: <issue>
  - Reason: <reason for flagging>
  - Clarification needed: <clarification needed>
```

If no issues: `No material transcription issues identified.`

### Output 3: Structured Transcript Digest

Requirements:
- Use headings; capitalise for readability
- Organise by topic
- Only create headings supported by explicit agenda items, repeated themes, or clearly grouped content
- Include ALL substantive information from the Concise Transcript
- Do not summarise away detail
- Do not add new information
- Do not introduce unsupported sections

This is a full-fidelity reorganisation, not a summary. Every substantive point from the Concise Transcript must appear at least once.

### Output Standards

- Be concise, faithful, and neutral
- Preserve nuance, uncertainty, disagreement, and open issues
- Prefer exact meaning over elegant phrasing
- Preserve ambiguity rather than resolving it
