# Plan: kusto-flow-action Skill

## Context

After running `/kusto-flow-run`, users need to drill into specific flow runs to see action-level detail (what each action did, outgoing HTTP calls, trace events, job history, storage operations). Currently this requires manually constructing complex KQL queries with 5 result sets. This skill automates that by picking representative flow runs from `/kusto-flow-run` output and fetching complete action lists.

## Deliverables

### 1. Skill Files

Create `C:\Users\chenlu\.claude\skills\kusto-flow-action\`:
- `SKILL.md` — Skill prompt (follows kusto-flow-run pattern)
- `references/queries.kql` — KQL query template with 5 result sets

### 2. SKILL.md Structure

#### Frontmatter
- name: `kusto-flow-action`
- description: Fetch flow run action details from Kusto. Requires `/kusto-flow-run` output. Use when user says "flow action", "kusto flow action", "action detail", "flow run detail", etc.

#### Workflow

**Step 1: Obtain inputs from session**

Retrieve from current session:
- `flowResourceKusto` from `/kusto-flow-summary` (contains `_environmentId`, `_flowName`, `_flowId`, `_serviceName`, `_cluster`, `_database`)
- Q2 FlowRunEvents result set from `/kusto-flow-run` (the full event data)

If either is missing, stop and ask user to run the prerequisite skill first.

**Step 2: Build criteria sets**

From the `/kusto-flow-run` Q2 result set (End events only), derive 4 criteria sets. Each criteria set picks one `flowRunSequenceId`:

| # | Criteria | Selection Rule |
|---|----------|---------------|
| 1 | Latest per status | For each distinct `status`: row with max `env_time` → take its `flowRunSequenceId`, `correlationId`, Start `env_time`, End `env_time` |
| 2 | Latest per statusCode | For each distinct `statusCode`: row with max `env_time` → same fields |
| 3 | Latest per status+statusCode | For each distinct `status`+`statusCode` combo: row with max `env_time` → same fields |
| 4 | Slowest | Row with max `durationInMilliseconds` → same fields |

Each criteria entry contains:
- `criteriaLabel` — e.g., "status=Failed", "statusCode=BadRequest", "status=Failed/statusCode=BadRequest", "slowest"
- `flowRunSequenceId`
- `correlationId` (from the End event row)
- `runStartTime` — matched Start event's `env_time` for that `flowRunSequenceId` (from Q2 data). If no Start event found, use End `env_time` - 1h.
- `runEndTime` — End event's `env_time`

**Step 3: Deduplicate by flowRunSequenceId**

Multiple criteria may resolve to the same `flowRunSequenceId`. Group criteria entries by `flowRunSequenceId`. Execute the Kusto query **once per unique `flowRunSequenceId`**. Track which criteria labels map to each `flowRunSequenceId`.

**Step 4: Load query template**

Read `C:\Users\chenlu\.claude\skills\kusto-flow-action\references\queries.kql`.

Single query body containing 5 `let` sub-queries that produce 5 result sets:
1. `_actionDetail` — ActionDetail (from TraceEvents)
2. `_outgoingRequests` — OutgoingRequest (from BJSJobHistory → CommunicationEvents join)
3. `_traceEvents` — TraceEvent (from BJSJobHistory → TraceEvents join)
4. `_jobHistory` — JobHistory (from BJSJobHistory)
5. `_storageEvents` — StorageEvent (from BJSJobHistory → BJSStorageEvents join)

**Step 5: Execute queries**

For each unique `flowRunSequenceId`, build preamble:
```kql
let _environmentId = '{value}';
let _flowName = '{value}';
let _flowId = '{value}';
let _serviceName = '{value}';
let _cluster = '{value}';
let _database = '{value}';
let _flowRunSequenceId = '{value}';
let _correlationId = '{value}';
let _runStartTime = datetime({runStartTime - 1h});
let _runEndTime = datetime({runEndTime + 1h});
```

Execute `preamble + query body` via `mcp__azure__kusto` with:
- cluster: `_cluster`
- database: `_database`
- tenant: `72f988bf-86f1-41af-91ab-2d7cd011db47`
- timeout: 180s, retry once on failure

**Step 6: Save output files**

For each criteria entry, generate one markdown file:
- **Filename:** `{_flowName}-flow-action-{sanitized-criteriaLabel}.md`
- If multiple criteria share the same `flowRunSequenceId`, duplicate the output (same content, different filenames per criteria label).

**Markdown file structure:**
```markdown
# Flow Run Action Detail

**Environment ID:** `{_environmentId}`
**Flow Name:** `{_flowName}`
**Flow ID:** `{_flowId}`
**Criteria:** `{criteriaLabel}`
**Flow Run Sequence ID:** `{_flowRunSequenceId}`
**Correlation ID:** `{_correlationId}`
**Run Start Time:** `{_runStartTime}`
**Run End Time:** `{_runEndTime}`
**Date:** {YYYY-MM-DD}

---

## 1. Action Detail ({count})
{formatted table — all rows, sorted by env_time asc}

## 2. Outgoing Requests ({count})
{formatted table — all rows, sorted by env_time asc}

## 3. Trace Events ({count})
{formatted table — all rows, sorted by env_time asc}

## 4. Job History ({count})
{formatted table — all rows, sorted by env_time asc}

## 5. Storage Events ({count})
{formatted table — all rows, sorted by env_time asc}
```

Also generate one TSV file per unique `flowRunSequenceId` per result set (5 TSV files):
- `{_flowName}-flow-action-{flowRunSequenceId}-actiondetail.tsv`
- `{_flowName}-flow-action-{flowRunSequenceId}-outgoingrequests.tsv`
- `{_flowName}-flow-action-{flowRunSequenceId}-traceevents.tsv`
- `{_flowName}-flow-action-{flowRunSequenceId}-jobhistory.tsv`
- `{_flowName}-flow-action-{flowRunSequenceId}-storageevents.tsv`

TSV files: header row + all rows, tab-separated, sorted by env_time asc. One set per unique flowRunSequenceId (not duplicated per criteria).

**Step 7: Declare session values**

```
Flow Action Detail: criteriaCount={N}, uniqueFlowRunSequenceIds={N}, queriesExecuted={N}, outputFiles=[list of .md filenames]
```

#### Error Handling
- Missing `/kusto-flow-run` output → stop, ask user to run it first
- Missing `/kusto-flow-summary` output → stop, ask user to run it first
- Query timeout → retry once at 180s
- Zero results for a result set → display "No data." in that section
- Auth failure → remind user about Microsoft corp tenant

### 3. queries.kql

Store the user-provided KQL as a single query template. Variables `_environmentId`, `_flowName`, `_flowId`, `_serviceName`, `_cluster`, `_database`, `_flowRunSequenceId`, `_correlationId`, `_runStartTime`, `_runEndTime` are provided via preamble.

The query uses `cluster(_cluster).database(_database).` prefix for cross-cluster queries. The 5 result sets are returned via the final `_actionDetail; _outgoingRequests; _traceEvents; _jobHistory; _storageEvents;` statement.

**Important:** Since `mcp__azure__kusto` connects to a specific cluster/database directly, and the query uses `cluster(_cluster).database(_database).` prefix internally, the connection cluster/database should match `_cluster`/`_database`.

### 4. Register skill trigger

Add to the available skills list. The skill name `kusto-flow-action` and its trigger phrases ("flow action", "kusto flow action", "action detail", "flow run detail", "run actions") should be registered.

## Files to Create/Modify

| File | Action |
|------|--------|
| `C:\Users\chenlu\.claude\skills\kusto-flow-action\SKILL.md` | Create |
| `C:\Users\chenlu\.claude\skills\kusto-flow-action\references\queries.kql` | Create |

## Verification

1. Run `/kusto-flow-summary` for a test flow
2. Run `/kusto-flow-run` to get flow run events
3. Run `/kusto-flow-action` and verify:
   - Criteria sets are correctly derived from flow-run sections 13-16
   - Deduplication works (same flowRunSequenceId → single query, multiple output files)
   - All 5 result sets appear in each output file
   - TSV files contain complete data
   - Time window is runStartTime-1h to runEndTime+1h

## Unresolved Questions

1. The user's KQL query uses `cluster(_cluster).database(_database).` prefix on every table access. Does `mcp__azure__kusto` support this when already connected to that cluster/database? If not, we may need to strip the prefix. Will verify during implementation.
2. Should there be a cap on total criteria entries (e.g., if a flow has 50 distinct statusCodes, that's 50+ criteria)? Suggest capping at 20 unique flowRunSequenceIds to avoid excessive queries.
