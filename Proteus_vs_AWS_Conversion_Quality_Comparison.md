# Proteus vs AWS Modernization Agent — Conversion Quality Comparison

Nov 3, 2026 · SF

Across the same 95 SQL Server routines, SourceFuse's Proteus agent produced 61 working PostgreSQL routines against the AWS Modernization Agent's 42 — 19 more delivered correct, and 25 fewer left needing rework.

## What each agent does

Both products exist to help move a database off SQL Server and onto PostgreSQL. They take on different amounts of that job.

**AWS SQL Server Modernization Agent** handles the migration end to end. It converts the database structure — tables, indexes, keys and constraints — moves the data across, and rewrites the stored procedures and functions that hold the business logic. One tool, the whole migration.

**Proteus**, built by SourceFuse, does one part of that job only: converting stored procedures and functions. It does not move tables or data. What it adds instead is a focus on correctness: a routine counts as complete only when it behaves the same as the original.

### The key difference

Breadth against depth.

The AWS agent covers more of the migration. Proteus covers less of it, but proves that the part it does is correct before handing it over.

That distinction matters because procedures and functions are the hardest part of a SQL Server migration and the easiest part to get silently wrong. Tables and data either move or they visibly fail. A converted procedure, by contrast, can compile cleanly, run without error, and still return different numbers — a defect that surfaces in production rather than in testing.

The rest of this report measures exactly that risk, on the one job both products do.

## What was compared

Both agents were given the same 95 stored procedures and functions from a public SQL Server database, the [Wide World Importers sample database for Microsoft SQL Server](https://learn.microsoft.com/en-us/sql/samples/wide-world-importers-what-is?view=sql-server-ver17), and asked to produce PostgreSQL equivalents. The converted code from each was then run through the same automated test suite, which executes every routine against both the original SQL Server database and the migrated PostgreSQL one and compares the results.

A routine counts as working only when its PostgreSQL output matches the SQL Server output exactly. Code that compiles but returns different data does not count.

In total the test run covered 1,380 test cases and compared 2,644 rows of output between the two databases:

| Schema | Test cases | Rows compared |
| --- | --- | --- |
| WebApi | 768 | 1,355 |
| Integration | 196 | 915 |
| Website | 135 | 320 |
| Application | 251 | 28 |
| Sequences | 30 | 26 |
| Total | 1,380 | 2,644 |

The two products do not have the same remit, and the comparison is deliberately narrow:

|  | Proteus (SourceFuse) | AWS Modernization Agent |
| --- | --- | --- |
| Converts procedures and functions | Yes | Yes |
| Converts tables, schema and data | No | Yes |
| Scope of this report | Full product | Routine conversion only |

The AWS agent covers a broader migration job. This report measures only the part both products do: turning SQL Server procedures and functions into PostgreSQL that behaves identically. It is not a verdict on the AWS agent's schema or data migration, which was not tested here.

## Headline results

Proteus produced working PostgreSQL code for 61 of 95 routines. The AWS agent produced working code for 42.

|  | Proteus | AWS | Difference |
| --- | --- | --- | --- |
| Routines attempted | 95 | 95 | same |
| Working after conversion | 61 | 42 | +19 |
| Pass rate, all routines | 64% | 44% | +20 points |
| Pass rate, testable routines only | 77% | 49% | +28 points |
| Still needing rework | 18 | 43 | 25 fewer |

The second pass rate excludes routines no test could fairly judge — those blocked by a missing dependency or with no observable output. On that basis, roughly three in four Proteus conversions worked against roughly one in two for AWS.

The gap in rework is the figure with the clearest cost attached. AWS left 43 routines requiring developer attention; Proteus left 18.

## Results by schema

*[Chart omitted from this export: Proteus and AWS test results · 95 routines across 5 schemas · November 2026. See the Word version or the live doc.]*

The gap is widest in Website and Application, where Proteus produced two to three times as many working routines. WebApi, the largest schema at 53 routines, is the closest contest. Sequences holds only two routines and both agents converted both.

## What the four outcomes mean

Every routine lands in exactly one of four categories. Only the first is a success; only the last is a failure.

| Outcome | What it means in practice | Proteus | AWS |
| --- | --- | --- | --- |
| Working | Runs in PostgreSQL and returns exactly what SQL Server returned. Ready to use. | 61 | 42 |
| Dependency missing | Could not be tested because something it calls was not available yet. Verdict pending, not a failure. | 10 | 6 |
| Non-testable | Has no observable output to compare, such as configuration and maintenance routines. Needs manual sign-off. | 6 | 4 |
| Needs rework | Ran, but produced the wrong result. Goes back for another conversion attempt. | 18 | 43 |

Two points of fairness are worth stating. Proteus had more routines it could not test — 16 against the AWS agent's 10 — which slightly narrows the base its 77% is measured on. And a routine marked "needs rework" is not permanently broken; it is work still queued.

Even allowing for both, the difference holds. The decisive number is 18 against 43: the routines a development team would have to pick up and fix by hand.

## What this means for delivery

The practical difference is 25 routines. That is the extra volume of hand remediation a team inherits when the AWS agent does this part of the job instead of Proteus.

This test did not measure how long a single remediation takes, so the figures below are illustrative arithmetic, not a measured result. They are included so stakeholders can apply their own rate.

| If one routine takes | Proteus rework (18) | AWS rework (43) | Extra effort |
| --- | --- | --- | --- |
| 4 hours | 72 hours | 172 hours | 100 hours |
| 8 hours | 144 hours | 344 hours | 200 hours |

Three consequences follow from the same number:

- **Schedule.** Remediation sits on the critical path. A larger backlog of failed routines pushes integration testing and user acceptance testing later.
- **Predictability.** At a 64% first-pass rate, most of the work is done and the remainder is visible. At 44%, more than half the estimate is still unknown.
- **Review load.** Every reworked routine needs a developer to read the original SQL Server logic, understand the intent and re-verify the fix. That is senior time, not routine time.

## What this comparison does not prove

The result is clear within its boundaries. The boundaries are worth stating plainly.

- **One database, one point in time.** 95 routines from a single SQL Server database, tested in November 2026. Both products continue to change, and a different workload could produce a different spread.
- **Correctness, not speed.** "Working" means the PostgreSQL routine returns the same data as the SQL Server original. How fast it runs was not measured.
- **Different remits.** The AWS agent also migrates schema and data, which was not assessed. A team choosing between them is not choosing between equals.
- **The test rig has its own failures.** In some runs, a share of failures traced back to faults in the test harness rather than the converted code. Those cases inflate the rework count for both agents, and not necessarily by the same amount.
- **Rework is a queue, not a verdict.** Routines marked for rework may pass on a later attempt. The figures describe first-pass quality, which is what drives schedule risk.

One open item should be confirmed before this report circulates widely: that both agents' output was run through the identical test harness and rule set. If the two runs used different configurations, the comparison weakens.

## Conclusion

On the work both products do — converting SQL Server procedures and functions to PostgreSQL — Proteus delivered 19 more working routines out of 95 and left 25 fewer for developers to fix. On a like-for-like basis among testable routines, 77% against 49%.

For organisations whose migration risk sits in stored procedure logic rather than schema movement, that difference is material to both cost and timeline.
