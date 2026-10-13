# Pluggable Boundaries

Research date: **2026-10-03**. Draft for review; not yet proposed on dev@.

This workstream asks one question: what clean pluggable boundaries should Apache Cassandra adopt so new ideas can grow outside the mainline, without forking and without disrupting the core? It catalogs every existing and proposed extension point, grades each one, and proposes an upstream roadmap.

It generalizes two earlier pieces of work: a pluggable-consistency prototype (Cassandra branch [`codex/pluggable-consistency-prototype`](https://github.com/pmcfadin/cassandra/tree/codex/pluggable-consistency-prototype) on a public fork) and a query-optimizer contract. Both are summarized in report 08, and both become slots in the proposed cuts. Their full write-ups live in a private research repository; report 08 is the public summary.

## Start here

**[ROADMAP.md](ROADMAP.md)** is the synthesis. It contains:

- **A ranked top 10 boundaries to create** (first section after the thesis).
- Design principles drawn from the project's history and the ecosystem (§2–3).
- The plugin platform that does not exist yet (§4).
- A catalog of about 100 extension points with status, grade, and recommended action (§5).
- The roadmap: **six cuts, many slots** (§6). The project does core surgery six times; every later idea plugs into an existing cut as an ordinary ticket.
- Mick's Cassandra mindmap mapped onto the cuts (§6a), and Josh McKenzie's practitioner input with the changes it caused (§6b).
- What not to do, open questions, and confidence notes (§7–9).

## Supporting reports

Each report carries the code citations (`file:line` against the `cassandra/` submodule) and URLs behind the roadmap's grades and claims.

| Report | Area |
|---|---|
| [01 Storage engine](reports/01-storage-engine.md) | Memtable, SSTable format, compaction, caches, commitlog, file system, journal |
| [02 Query and indexing](reports/02-query-indexing.md) | Index API, SAI, vector, query handler, functions, types, virtual tables, planning |
| [03 Distribution and consistency](reports/03-distribution-consistency.md) | Replication, placement, TCM, Accord, coordinator paths, repair, streaming, messaging |
| [04 Security and operations](reports/04-security-ops.md) | Auth, audit, encryption, observability, guardrails, config, nodetool |
| [05 Edges, protocol, modularity](reports/05-edges-protocol-modularity.md) | Native protocol, internode wire, serializers, CDC, offline tools, build layering |
| [06 Upstream history](reports/06-upstream-history.md) | About 60 CEPs, JIRA and dev@ positions on pluggability, analogous projects |
| [07 Ecosystem](reports/07-ecosystem.md) | Sidecar, Analytics, Management API, drivers, third-party plugins and their upgrade pain |
| [08 Prior research](reports/08-prior-research.md) | Lessons from this repo's own pluggable-consistency and optimizer work |

ROADMAP §6a cross-checks the roadmap against an Apache Cassandra feature mindmap shared by Mick Semb Wever. The mindmap itself is not published here; §6a lists every idea on it.

## How to review

- **Facts:** code citations (`file:line`) are against Apache Cassandra trunk at [`0691eb3319`](https://github.com/apache/cassandra/tree/0691eb3319e5fc33a8d393f48120bb7b3ca901da). Paths written as `cassandra/...` are relative to that source tree. Items that exist only on the prototype branch are marked as such.
- **Not verified:** the analogous-project section of report 06 was written from memory. Each report ends with its own open items.
- **Not run:** no builds or tests were run for this research. The Accord-writes-skip-CDC finding (report 05) comes from reading code and has not been reproduced.
- **Open decisions** are in ROADMAP §8. The two that matter most:
  - Will dev@ accept the request-pipeline cut as one staged CEP?
  - How should "expressive consistency levels" from the mindmap be reconciled with keeping `ConsistencyLevel` closed?
