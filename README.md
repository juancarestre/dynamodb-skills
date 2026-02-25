# DynamoDB Single-Table Design Skills

Agent skills for designing DynamoDB single-table schemas through an interactive 3-step pipeline.

## Skills

| Skill | Description |
|---|---|
| `dynamodb-single-table` | Orchestrator — detects pipeline state and routes to the right step |
| `dynamodb-access-patterns` | Step 1: Elicit entities, attributes, and access patterns interactively |
| `dynamodb-table-design` | Step 2: Design PK/SK/GSIs and map every access pattern to a DynamoDB operation |
| `dynamodb-query-interfaces` | Step 3: Generate repository interfaces, key builders, and entity models |

## Pipeline

```
Access Patterns  -->  Table Design  -->  Query Interfaces
   (Step 1)            (Step 2)            (Step 3)
```

Each step produces a `.md` file that feeds into the next. By the end you have a complete specification for your data access layer.

## Install

```
npx skills add anomalyco/dynamodb-skills
```

## Usage

Start a conversation with your agent and say:

> I need to design a DynamoDB table for my application

The orchestrator skill will detect your pipeline state and guide you through the right step.

## Example

The [`example/`](example/) folder contains the full output of a real pipeline run for a **Music Ranking** app (multi-user top 100 with genre/artist filtering and global rankings):

| File | Description |
|---|---|
| [`access-patterns.md`](example/access-patterns.md) | 4 entities, 16 access patterns |
| [`table-design.md`](example/table-design.md) | 2 GSIs, inverted score sort keys, 5 item types |
| [`query-interfaces.md`](example/query-interfaces.md) | TypeScript interfaces, 17 key builders, 3 repositories |
