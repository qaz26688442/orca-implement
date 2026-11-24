# Child notes

Write `<feature>/implement-notes.md` with these sections, in the spec's language and filled in for this repo and spec. Every child reads it before its ticket, so it carries what a fresh session cannot find by looking.

- **Where you work.** Give the worktree path and branch. The child commits only there. Merging, pushing and the main checkout belong to the acceptor.
- **Read first.** List the spec, the child's ticket, the "Implementation notes" and "Acceptance" sections of earlier tickets, the repo glossary, and the ADRs the spec cites.
- **Conventions.** Include only the repo rules a newcomer gets wrong: i18n, which UI components to reuse, migration numbering, and the ADR number reserved for this spec. Tests reach paid or external services only through fakes.
- **Tests.** Give the exact isolated test command, the suites to run for a ticket, the repo's rule check, the lint command, and the environment gotchas that cost earlier sessions time. Then state the testing rules the child writes to, with `CODING_STANDARDS.md` (Testing) as their source:
  - each test sits at a row of the spec's test table;
  - the variants at one seam are one table-driven test, its preparation built once;
  - a test beyond the table is justified in the implementation notes;
  - preparation comes from `conftest.py` or a shared fixture module, never from another test module;
  - screenshots and page checks come from a one-off acceptance script in the feature's `acceptance/` folder;
  - the rule check passes.
- **Baseline.** List the failures at the base commit. The acceptor fills this in.
- **Before you finish.** The child works through these in order:
  1. Run the affected suites and the rule check, and show no failure beyond the baseline.
  2. Run the code-review skill on the diff since the dispatch, and fix the confirmed findings.
  3. Append `## Implementation notes (<date>)` to the ticket: what was built, the tests added and the seconds they cost (Testing T17), deviations and their reasons, judgement calls, and items left for a human. Tick the criteria that are met.
  4. Commit.
  5. Send `worker_done` as the preamble says. Its summary names the commit hashes and the test counts (passed and failed, and which failures are baseline). Then idle.
- **Questions.** A decision that changes user-visible behaviour or stored data goes to the acceptor through the preamble's `ask`. The child decides technical choices itself and records them in its implementation notes.
