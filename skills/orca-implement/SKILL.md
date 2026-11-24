---
name: orca-implement
description: Run a spec's tickets one at a time through supervised Orca child sessions that build each ticket with /implement, while this session accepts each before the next.
disable-model-invocation: true
---

# Orca implement

Run the tickets `/to-spec` and `/to-tickets` produced as a **relay**: one leg per ticket. A fresh child session runs each leg with the implement skill, and the baton passes only when this session, the **acceptor**, has accepted the leg. The acceptor dispatches, answers, accepts and merges. Every code change comes from a child.

Argument: the spec folder or ticket paths. Without one, use the feature folder this conversation just produced.

## 0. Load Orca

Resolve the Orca CLI the way the `orchestration` skill says, then read `ORCA skills get orchestration` and the references it names for new worktrees and launch preferences. That guide is the source of truth for every Orca command. The flags below are only the choices this skill makes.

Done when `ORCA status --json` reports the runtime ready.

## 1. Choose the child agent

| Situation | `worker-start` flags |
|---|---|
| The user named an agent, model or effort | Exactly what they named. Fill the parts they left out from that agent's row below |
| This session is Claude Code | `--agent claude` with no `--model`, so the child inherits the user's default |
| This session is Codex | `--agent codex --model gpt-5.6-sol --effort medium` |
| This session is Antigravity (`agy`) | `--agent antigravity` with no `--model`, so the child uses the model in Antigravity's settings |

An Antigravity child reads skills from `~/.gemini/config/skills/` and the project's `.agents/skills/`, not from `~/.agents/skills/`. Before its first leg, check that `implement`, `tdd`, `code-review` and `orchestration` are there; if one is missing, stop and tell the user how to install it (README of this skill's repository).

Done when the choice is in the child notes and you have told the user in one line.

## 2. Prepare the track

1. **Frontier.** Read the spec and every ticket, and list the frontier: tickets whose blockers are all accepted. A ticket that needs human input (a paid run, credentials, real data, a budget) is the acceptor's leg. Ask the user before running it.
2. **Worktree.** Create one Orca worktree for the whole spec from the default branch (`ORCA worktree create --name <feature-slug> --base-branch <default>`). Every leg shares it and stacks commits on its branch.
3. **Test isolation.** When the repo's tests share a database or service with other sessions, give this worktree its own, for example a settings override that names a separate database. Run one test run at a time in the worktree, so the acceptor's runs and a child's runs never overlap.
4. **Child notes.** Write `<feature>/implement-notes.md` from [CHILD-NOTES.md](CHILD-NOTES.md) and commit it. Then run the affected tests once at the base commit and record the failures in the notes as the **baseline**.
5. **Run.** Bind one Run: `ORCA orchestration run-create --objective "<spec>: tickets as a relay"`.

Done when the worktree sits at the base commit with the notes and baseline committed, and the Run is bound.

## 3. Run each leg

Take the next frontier ticket:

1. **Dispatch.** `ORCA orchestration worker-start --spec "<leg spec>" --worktree id:<worktree id> --task-title "<NN title>" <child flags> --json`.
   - The leg spec follows Orca's task-spec contract.
   - It opens by invoking the implement skill on the ticket and the notes: `/implement <ticket> <notes>` for Claude and Antigravity, `$implement <ticket> <notes>` for Codex.
   - It also names the skill file where the child's agent reads it (`~/.agents/skills/implement/SKILL.md` for Claude and Codex, `~/.gemini/config/skills/implement/SKILL.md` for Antigravity), so the child follows the skill even when the invocation syntax does not fire.
   - Record the base commit of the dispatch, and run the existing test files the ticket's seam rows point at on it. Keep that run summary as the leg's **before-summary**.

   Done when the call exits 0 and `launch.effective` shows the chosen agent and model. On a mismatch, stop the relay and tell the user.
2. **Wait.** Run `ORCA orchestration check --wait --types worker_done,escalation,question` in a background shell, and re-arm it after each timeout.
   - Answer technical questions yourself.
   - Take a question that changes what users see or what data is kept to the user, and pass their answer back with `reply`.
3. **Accept.** When `worker_done` arrives for this dispatch, run [Acceptance](#acceptance).
4. **Settle.** Release the worker, or reuse its terminal for a fix-up. Then acknowledge the delivery, tell the user the leg's verdict in one line, and take the next ticket.

## Acceptance

Base the verdict on evidence you produce yourself. The child's report is a claim, not evidence. The repo's testing rules (`CODING_STANDARDS.md`, Testing) are the standard for the tests the leg wrote.

- **Criteria.** Review the diff from the dispatch base to the branch tip against every acceptance criterion of the ticket and the spec's decisions. Map each criterion to evidence: a test at a row of the spec's test table, a command output, or an acceptance script's output.
- **Tests.** Run the affected tests and the repo's rule check yourself in the worktree. They show no failure beyond the baseline, and lint is clean on the changed files.
- **Seen running.** When the ticket changes a page, look at that page through a browser or the ticket's acceptance script, run once (Testing T5). Screenshots come from that script; the suite keeps only the browser tests that Testing T7 allows.
- **Cost.** Count the tests the leg added from the diff, by file, and the seconds the suite gained: run the before-summary's selection again, plus the leg's new test files, and compare the two summaries (Testing T17). When the comparison says the runs are not comparable, record that and its reason in place of seconds.
- **Scope.** The diff holds only this ticket's work, and every edit is inside the worktree.

Append `## Acceptance (<date>)` to the ticket, in the spec's language. Record the verdict, what you ran and its counts, the tests added and the seconds gained, the findings, and any decisions handed to the user. Commit it on the branch.

A rejected leg goes back with its findings, either to the same worker (`send --to dispatch:<id>`) or to a new fix-up leg. Then accept again from the top.

## 4. Finish

Done when:

- every ticket is accepted or explicitly deferred by the user;
- the full test suite has run once on the branch tip;
- the feature's new tests, found in the diff since the base commit, are inventoried as keep, merge or retire, each with its reason (Testing T11–T13); the user has read the retire list, and a child leg has merged or deleted only what the user agreed to;
- the full suite's time is set against the budget (Testing T16), and a breach comes with a proposed cleanup ticket that the user takes or leaves before the next spec;
- `worker-list --run <run id> --terminal-state reclaimable` returns none.

Report each ticket's verdict, commits, evidence, tests added and seconds gained, and open items, then the inventory and the budget result. Merging into the default branch waits for the user's go-ahead: fast-forward from a clean checkout. A checkout where another session has uncommitted changes to the same files is not clean.
