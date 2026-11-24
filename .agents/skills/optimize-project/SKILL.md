---
name: optimize-project
description: >-
  Use when Delivery Lab needs several independent optimizations or fixes and the user asks for a lead agent to coordinate parallel subagents, including “并行优化”, “主代理统筹”, or “分派子代理”.
---

# Optimize Delivery Lab

The lead agent coordinates the batch in the current ZCode task. Delegate investigation, workspace preparation, implementation, testing, review, documentation and integration to the agents below. The lead owns scope, scheduling, reservations, handoffs and the final report.

Apply the project constraints in `AGENTS.md` when present. Keep Delivery Lab aligned with Clowder AI for product behavior and DeepSeek Harness for runtime and plugin design. Preserve the desktop and zero-build frontend goals. Ask scouts to trace reference source and tests and identify revisions. Use `ROADMAP.md`, `docs/known-issues.md` and current reports to prioritize evidenced gaps in user outcomes, correctness and recovery. Define the normal entry point, expected result and failure or recovery checks for each chosen improvement. [Clowder AI](https://github.com/zts212653/clowder-ai) · [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness)

## Model and agents

Select **`magpie/codex/gpt-6.2-sol` with `ultra`** for the lead in the ZCode model picker before invoking this skill. Verify the current selection through trusted host metadata or the user's confirmation when the tool only shows supported levels. Every role uses the same model and effort in its `.zcode/agents/` definition.

| Native `Agent` type | Assignment |
|---|---|
| `optimization-scout` | Read-only investigation, reference comparison and independent review |
| `optimization-worker` | One approved implementation, test, documentation or acceptance assignment in its own worktree |
| `optimization-integrator` | Prepare worktrees, assemble and verify the batch, apply it to the shared checkout, and commit and push to GitHub |

Use only these explicit `subagent_type` values. Check that ZCode has loaded the matching definitions before dispatch and check the effective route when the host reports it. If this task predates the definitions and the types are missing, use a freshly opened Delivery Lab task. A missing role or mismatched setting blocks that dispatch. Keep every child and its approvals in the current task. Use `SendMessage` to continue a known child. Keep its assignment reserved until the host confirms it has stopped writing.

## Coordinate the batch

1. **Discover in parallel.** Give scouts separate questions within the user's scope. Each returns the proposed outcome, source and test evidence, prerequisites, write paths and mutable resources. Choose a bounded batch from their findings. For a planning-only request, return the ranked findings and stop there.
2. **Claim the work.** Verify the checkout root and full current task ID. Use the `.zcode/continue-state/<session-id>.md` registry shared with `continue-project`. Before dispatching writers, atomically record the slice, status, write paths, mutable resources, checkout and `HEAD` using a temporary file and rename. Reread the other checkpoints and the worktree. Treat overlapping slices, path prefixes or resources in `reserved`, `working` or `blocked` checkpoints as occupied and preserve other worktree changes. Repeat the check before widening the claim. Keep checkpoints local.
3. **Assign and dispatch.** Ask the integrator to capture a common baseline and prepare one detached Git worktree per writer or mutable acceptance run, plus one for integration. Pass each child one outcome, dependencies, the absolute worktree path, exclusive write paths, runtime resources, acceptance checks and the handoff destination. Keep every ready, independent assignment running within host concurrency limits. Give each file one writer per batch, including shared docs and generated files. Group coupled changes under one writer, or publish and recapture the baseline before dispatching a dependent batch. Serialize mutable resources that cannot be isolated. Run read-only reviews as soon as the required evidence is ready.
4. **Integrate and verify.** Give one integrator the worker patches and reports. The integrator validates their paths, combines them in isolation and runs the applicable checks. Scouts can review the combined result in parallel when they only read it. Send defects back to the responsible child. Use `pnpm check`, `pnpm test` and `pnpm format:check` as applicable. Delegate changed desktop journeys and failure or recovery checks. Get explicit user authorization for real provider calls.
5. **Commit and push.** The user has authorized publication to GitHub `main` for every completed optimization task. After acceptance, have the integrator prepare a commit for the task's verified changes, apply the patch to the shared checkout and push the commit to `refs/heads/main` on the configured GitHub remote. Verify that `main` contains the commit before marking the task `completed`. One verified push can cover several completed tasks in the same batch. Honor a newer user request to hold publication. For a task with no changes, record the result without an empty commit.
6. **Close the batch.** Record the acceptance evidence and the verified GitHub `main` commit SHA. Use the integrator's publication record to tell the user exactly what was pushed. If publication fails, save the local commit, apply status and reason as pending and retry from that checkpoint. Release claims after the children and services using them have stopped and record any paths still owned by unfinished work.

## GitHub publication

Use a separate publication worktree on a local branch based on the latest verified GitHub `main`. Include only the verified task changes and their in-scope prerequisites. If the patch relies on unpublished work outside the task scope, report the prerequisite for coordination. Push with an explicit refspec to `refs/heads/main` using a normal fast-forward push. If GitHub `main` advances, verify whether it already contains the task commit. If publication is still pending, use the new base to integrate and verify the task again before retrying.

Give each commit a subject that names the outcome and a body that explains the concrete changes and why they were made. After each verified push, report the GitHub repository and `main`, the commit SHA and link, the changes and affected paths for each task, and the checks run with their results. Use the exact published range for the file list and link its GitHub comparison when the push spans multiple commits. Report pending work separately with the reason.

## Checkout coordination

Before reserving a write batch, acquire one coordinator slot in the checkout. With the verified `task_id` and absolute checkout `root`, create the owner link atomically:

```sh
state="$root/.zcode/continue-state"
mkdir -p "$state" && ln -s -n "$task_id" "$state/.optimize-project-owner"
```

If acquisition fails because another owner exists, read the owner and keep to read-only discovery until the slot is free. An owner can resume its own batch after checking the prior children and reservations. After all writes stop and the checkpoint is saved, the owner can `unlink` the exact link after verifying `readlink` still returns its own task ID. For an abandoned slot, verify that the owner and its children have stopped and record the unfinished claims before a user-authorized handoff. A stale-looking timestamp or changed `HEAD` leaves ownership in place. Shared resources outside a worktree require their own isolation or serialized use.

**Example.** A scout finds a timeline rendering defect and another finds a CLI recovery defect. Their workers get different worktrees, paths, data directories and ports. The integrator validates both patches and runs the combined checks. If both fixes need a change to the same session contract, group that change under one owner and release dependent work after the contract is settled.
