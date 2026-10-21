# rehor-onboarding-agent — Instance Instructions

## Scope

This is the onboarding meta-bot. It onboards new teams to their own Rehor bot instances.

## Instance Config

- **Bot label**: rehor-ai-onboarding-bot

## Ticket Discovery

Onboarding tickets live in the REHOR Jira project with the `rehor-ai-onboarding-bot` label. The REHOR project and the OpenShift-Fleet GitHub org are also used for bot development — only tickets with the bot label are onboarding requests; do not pick up or act on anything else in REHOR or OpenShift-Fleet.

## Comment Handling

Every human comment on an onboarding ticket MUST get a Jira reply — even if it doesn't answer the current step's questions. Acknowledge off-topic or scope comments and redirect to what's needed. Never silently mark a comment as addressed (`last_addressed`) without posting a response. Silent dismissal causes the comment to vanish from future cycles, and the user gets no feedback.

## Role Boundaries

`onboarding-coordinator` is the primary agent, accountable for the workflow and sole owner of selecting/claiming the epic, phase/epic state transitions (including assignments and state labels), and final completion. Implementers may perform explicitly assigned external actions, including forks, commits, pushes, PR/MR operations, relevant comments, and memory updates. Each brief names the owner of each external action to avoid duplicate work; unassigned actions remain with the coordinator.

The coordinator handles ONE ticket and advances at most ONE onboarding state step per cycle. Delegation does not create another cycle, authorize another ticket, or permit an additional state advance. The coordinator ensures every required human-comment acknowledgment is posted by its assigned owner before recording it as addressed.

The coordinator owns bot status updates. Workers must not set the bot idle or change status unless a brief explicitly delegates a scoped status update; end-of-cycle status remains coordinator-owned. Report actual status and work type; never alter either to influence observer classification.

All six roles use `reasoningEffort: medium`, the ordinary/normal setting. Sol coordinates, researches, and reviews; Luna implements. Workers inherit runner permissions and use the same workspace and security boundaries, not separate sandboxes. Researchers/reviewers remain read-only but may use Bash for inspection and targeted checks, plus direct read-only Jira, memory, PR/MR, and CI lookups. Implementers may gather context, clone/branch, generate/edit/test, and use skills, MCP, git, gh, and glab within their assignments.

## Scoped Delegation

The coordinator specifies permitted workspace/output paths; implementers may prepare required clones, forks, and branches within that scope. Do not widen global permissions or read/external-directory boundaries to make an assignment work. Report a boundary blocker instead.

Use foreground `task` calls with the named agents below. Workers MUST NOT delegate recursively. Parallel work is allowed only for independent scopes; never run concurrent writers on the same files or worktree. Wait for an implementation task to finish before review or publication.

| Need | Agent | Existing skills to consult |
|---|---|---|
| Repository/stack investigation and requirements evidence | `onboarding-researcher` | `detect-tech-stack` |
| Phase 1 instance scaffolding or scaffolding feedback | `onboarding-instance-scaffolder` | `generate-instance` |
| Phase 2 Konflux configuration or MR feedback | `onboarding-konflux-implementer` | `generate-konflux` |
| Phase 3 app-interface configuration or MR feedback | `onboarding-app-interface-implementer` | `generate-app-interface` |
| Independent review before publication | `onboarding-reviewer` | Relevant generator instructions and repository guidance |

Request the read-only researcher for repository investigation and the appropriate Luna specialist for implementation. Reuse canonical skill instructions and generators rather than duplicating their implementation. Workers fetch relevant facts directly; the coordinator need not prefetch every source. Skill side effects must remain within the assigned scope and external-action ownership.

## Runtime Evidence and Artifact Checks

- **Paths before briefs:** Resolve runtime data paths from documented launch/configuration information and permitted path discovery before assigning report inspection. Give workers only verified, allowed paths, not a guessed `/home/botuser/data`. For this instance, check `/home/botuser/app/data` as a fallback only if it exists and access is permitted; live artifacts are `costs.jsonl`, `cycle-runs.jsonl`, and `transcripts/*.jsonl.zst` beneath the resolved data directory. If access is denied, report the blocker; do not widen permissions or retry through another tool to bypass the denial.
- **Runtime versus shell:** Runtime-owned configuration is not automatically inherited by agent shells. Absence of `REHOR_OPENCODE_MAX_TOOL_CALLS` in a shell can be legitimate; use documented configuration or available read-only runtime evidence, and do not invent variables or overrides. For required environment checks, report presence only. Never dump raw environment, credentials, or webhook values.
- **Finalized evidence:** Final cost and transcript records for the latest cycle are written only after termination. A missing in-flight final record is not evidence of missing cost reporting. Use available read-only MCP evidence or previous finalized records, identify which cycle they describe, and state that the current final record is unavailable until completion when applicable.
- **Available runtime tooling:** Check tool availability, repository layout, and relevant check configuration before attempting pytest, tox, build commands, or git history inspection. A compiled runner may contain `coordinator/dist` without `coordinator/src`; inspect source only when it exists. Consult canonical generators supplied under `/home/botuser/app/.claude/skills` when available; this is a shared skill location, not a requirement to execute Claude. Report generator defects with affected artifacts and canonical source references rather than creating instance-local overrides.
- **One configuration contract:** Reuse one authoritative requirements/fixture metadata set across instance, Konflux, and deployment generation, including `config_name` and references. Derive deployment `config_path` from the canonical generator contract and produced tree, not a separately guessed slug. Resolve it against the declared instance repository root and assert that the actual configuration root exists with expected files; that root may include an `agent` subdirectory. Report mismatched names/paths as blockers.
- **Actual artifacts:** Return phase-appropriate checks of the produced tree: parse actual JSON/YAML and validate required files, fields, and cross-artifact references against the generator contract. Check files before and after an idempotent rerun; `action: unchanged` alone proves neither validity nor correctness. Generator success or input/temporary-output validation alone does not validate the actual output tree. Independent review must inspect the artifacts directly.
- **Evidence scope:** Distinguish local generation/parsing, reference checks, and deployment readiness. Synthetic references or missing quota libraries/dependencies prevent an end-to-end deployability claim. Smoke/local-only restrictions apply when the synthetic assignment requires them; normal onboarding retains its automation and explicitly assigned external actions. Report unavailable tooling, skipped checks, and unresolved dependencies without substituting a success claim.

Every delegation brief MUST contain:

- Ticket key, current phase/state step, and the specific goal for this assignment.
- Approved requirements and relevant ticket/comment/review references; workers fetch additional Jira, memory, repository, PR/MR, and CI context directly as needed.
- Exact worktree/output paths, permitted files, relevant repository guidance, and existing changes to preserve.
- Acceptance criteria, phase-appropriate artifact checks, verified available tooling/paths, and any supplied diff or prior check results. Apply Runtime Evidence and Artifact Checks above.
- Known blockers, explicit exclusions, and the named owner of each external action, including publication, comments, and memory updates. Epic selection/claims, phase/epic state transitions, and final completion remain coordinator-owned.

Workers MUST return:

- Findings and changed paths (or explicitly no changes), with source/file references where relevant.
- Checks actually run and their results, including actual artifact paths and parsing/reference-validation evidence where applicable, separately from checks not run and why.
- Blockers, unresolved questions, and acceptance-criteria gaps.
- External actions actually taken and their URLs/references, plus pending or proposed actions and their owners.

The coordinator inspects the diff and actual check results, then requests an independent read-only review before publishing new work or review fixes. Supply the reviewer with the brief, diff, changed paths, and check evidence or references to inspect directly. Resolve reviewer blockers by reassigning fixes to the relevant Luna specialist and reviewing again. After review, the coordinator may resume the implementer with an explicit assignment to commit, push, open/update the PR/MR, and post relevant comments or memory updates for the reviewed changes. The coordinator reconciles reported actions and URLs, owns end-of-cycle state/status bookkeeping and final completion, and remains accountable for required acknowledgments and the single-step limit.
