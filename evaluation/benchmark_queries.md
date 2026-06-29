# Benchmark Query Set — Structure

50+ queries across 5 categories used to evaluate agent accuracy.
Actual pipeline data is proprietary. Categories and representative examples below.

## Categories

### 1. Failure Diagnosis (18 queries)
- "Why did the [layer] load fail on [date]?"
- "What caused the timeout in the [pipeline] run?"
- "Which step failed in yesterday's SAP ingestion?"

### 2. Lineage Tracing (12 queries)
- "Which pipelines wrote to [table] this week?"
- "What is the upstream source for [Gold table]?"
- "Did [table] get updated in the last 24 hours?"

### 3. Live Status (8 queries)
- "Is [pipeline] currently running?"
- "What is the last successful run time for [pipeline]?"

### 4. Schema Metadata (7 queries)
- "What columns does the [table] Silver table have?"
- "What is the data type of [column] in [table]?"

### 5. Out-of-Scope / Refusal (5+ queries)
- Queries with no pipeline context → agent must refuse, not hallucinate

## Scoring
- Correct: Answer matches known ground truth
- Refused correctly: Out-of-scope query handled with "insufficient context" response
- Incorrect: Wrong answer provided (targeted for root-cause analysis)

**Overall result: 85%+ correct or correctly refused**
