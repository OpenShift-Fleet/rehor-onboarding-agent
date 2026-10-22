# rehor-onboarding-agent

Rehor bot instance for onboarding new teams to their own bot instances, built on [dev-bot](https://github.com/OpenShift-Fleet/rehor).

## How to onboard your team to Rehor using this agent

1. **Create a Jira Epic** in your team's Jira project with a summary like "Onboard \<team name\> to Rehor"
2. **Apply the label** `rehor-ai-onboarding-bot` to the epic
3. **Wait for the bot** — it runs on a cron schedule (weekdays 9am–6pm ET) and will pick up your ticket automatically
4. **Follow the bot's instructions** — it will comment on the epic asking for details (instance name, target repos, workflow type, etc.) and guide you through three phases:
   - **Phase 1: Instance Setup** — bot scaffolds your instance repo and opens a PR
   - **Phase 2: Konflux CI/CD** — bot generates Konflux config and opens an MR
   - **Phase 3: Deployment** — bot generates app-interface config and posts final manual steps

The bot tracks progress using `onboarding:*` labels on your epic. Reply to its comments in Jira to provide information or confirm steps — it will advance automatically.

## Developer Setup

```bash
git clone --recurse-submodules https://github.com/OpenShift-Fleet/rehor-onboarding-agent.git
cd rehor-onboarding-agent
```

## Submodule Update

This native V2 migration pins `dev-bot` to **`e6b0fda738c71b8be694b8711cf32e53eee59c9b`**, the merged commit from [Rehor PR #582](https://github.com/OpenShift-Fleet/rehor/pull/582). Initialize the recorded gitlink rather than following the remote branch:

```bash
git submodule update --init --recursive dev-bot
git -C dev-bot rev-parse HEAD
```

Review future gitlink updates with the matching configuration and image changes before recording a new pin.

## Native OpenCode configuration

- [`instance/rehor-onboarding-config/agent/instance.yaml`](instance/rehor-onboarding-config/agent/instance.yaml) selects the `onboarding` workflow, native V2 `opencode` runtime, `rehor-openai` provider, and `gpt-6.1-sol` model.
- [`agent/AGENTS.md`](instance/rehor-onboarding-config/agent/AGENTS.md) supplies instance guidance alongside native core and onboarding workflow `AGENTS.md` instructions.
- [`agent/opencode.json`](instance/rehor-onboarding-config/agent/opencode.json) selects `onboarding-coordinator` as the default primary agent and sets `experimental.subagent_depth: 1`. Native role files live under [`agent/.opencode/agents/`](instance/rehor-onboarding-config/agent/.opencode/agents/).
- [`opencode-deployment.json`](instance/rehor-onboarding-config/opencode-deployment.json) uses native provider `package: "@opencode/ai/providers/openai/responses"` and `settings`. This Responses provider ships with pinned CLI **2.1.2**; `packages: []` requires no separate AI SDK package metadata. Both Sol and Luna declare `capabilities.reasoning.supported: true` and model `settings.reasoningEffort: "medium"`. The native adapter lowers that setting to wire `reasoning.effort`; the deployment field is not `settings.reasoning.effort`.
- All six agents also explicitly set `request.body.reasoning.effort: medium`. Their ordered native `permissions` preserve the coordinator's five-worker `subagent` allowlist, all workers' delegation denials, and researcher/reviewer edit denials. Legacy `options`, permission maps, and top-level delegation depth are not translated.
- Deployment-owned `additionalDirectories` retains exactly `/home/botuser/repos`, `/home/botuser/data`, and `/tmp/opencode`. The runner renders a global native V2 policy ceiling; project or agent grants cannot widen it. Ordered agent restrictions remain effective within approved roots. These grants configure OpenCode permissions, not filesystem mounts or a Bash sandbox.
- [`agent/mcp.json`](instance/rehor-onboarding-config/agent/mcp.json) explicitly enables Jira `codemode: true`. Other generated MCP servers, including memory, retain native default `true` when omitted. Allowed active generated wrapped MCP grants automatically provide inherited `execute` access; agent permissions can further restrict it. Native Code Mode accounting counts each successful nested operation once, including recovered evidence; failed operations and the outer wrapper consume no tool-call budget unit. Batch limits are enforced at the next root-model boundary, not atomically per nested call.
- Keep the `_none` env sentinel for `install-envs.sh` (build) and `entrypoint.sh` (runtime): an empty env list currently skips env installation but makes the runtime run every preset entrypoint.

The locally inspected app-interface source, `data/services/insights/platform-frontend-ai-dev/deploy.yml`, already wires the `rehor-onboarding-agent` target to:

| Parameter | Configured value |
|---|---|
| `BOT_CONFIG_REPO` | `https://github.com/OpenShift-Fleet/rehor-onboarding-agent.git` |
| `BOT_CONFIG_PATH` | `instance/rehor-onboarding-config` |
| `BOT_EXECUTION_ENGINE` | `coordinator` |
| `REHOR_OPENCODE_DEPLOYMENT_CONFIG` | `/home/botuser/app/instance/rehor-onboarding-config/opencode-deployment.json` |
| `REHOR_OPENCODE_MAX_TOOL_CALLS` | `200` |
| `REHOR_MODEL_PROXY_URL` | `http://devbot-proxy:8450/v1` |

[`deploy/template.yaml`](deploy/template.yaml) requires these parameters and `REHOR_MODEL_PROXY_TOKEN` without defaults. The token is a non-secret client placeholder; the real OpenAI API key stays on the shared proxy. The coordinator syncs the remote profile for new cycles; the baked `instance/` directory is not a runtime profile fallback. Provider configuration selected by `REHOR_OPENCODE_DEPLOYMENT_CONFIG` is baked into the image and requires a rebuild to change.

The same local GitOps source already includes **both `gpt-6.1-sol` and `gpt-6-luna`** in shared-proxy `OPENAI_ALLOWED_MODELS`. No new parameter or allowlist edit is needed for this migration. This is source inspection, not confirmation of live reconciliation. Preserve other shared allowlist entries; `VERTEX_ALLOWED_MODELS` does not authorize OpenAI routes. This OpenAI-only onboarding template and target do not declare GCP/Vertex parameters; the separately deployed shared proxy retains its configuration for other bots.

See the shared [native V2 migration contract](dev-bot/docs/migrations/opencode-migration.md#profile-configuration-reference) for configuration, Code Mode, policy precedence, and validation details.

### Onboarding agents

| Agent | Model | Reasoning effort | Responsibility |
|---|---|---|---|
| `onboarding-coordinator` | `rehor-openai/gpt-6.1-sol` | `medium` | One ticket/one state step per cycle; accountable for delegation, epic selection/claims, phase/epic state transitions, and final completion |
| `onboarding-researcher` | `rehor-openai/gpt-6.1-sol` | `medium` | Read-only repository/stack investigation |
| `onboarding-instance-scaffolder` | `rehor-openai/gpt-6-luna` | `medium` | Phase 1 context, generation, edits, checks, and explicitly assigned publication |
| `onboarding-konflux-implementer` | `rehor-openai/gpt-6-luna` | `medium` | Phase 2 Konflux context, changes, checks, and explicitly assigned publication |
| `onboarding-app-interface-implementer` | `rehor-openai/gpt-6-luna` | `medium` | Phase 3 app-interface context, changes, checks, and explicitly assigned publication |
| `onboarding-reviewer` | `rehor-openai/gpt-6.1-sol` | `medium` | Independent read-only review before publication |

The coordinator supplies scoped briefs with workspace/output paths and a named owner for each external action, uses foreground tasks, and requests independent review before publication. After resolving review blockers, it may resume the implementer to publish reviewed changes and perform explicitly assigned relevant comments/memory updates. Workers gather context directly through Jira/memory/PR/MR/CI lookups; implementers may clone/branch and use skills, MCP, git, gh, and glab for assigned work. Researchers/reviewers retain edit denial and read-only scope, with Bash available for inspection and targeted checks. Workers report actions actually taken and their URLs; none may recursively delegate. The coordinator remains accountable for every required human-comment acknowledgment before recording it as addressed, end-of-cycle state/status bookkeeping, and the single-step limit. These roles inherit runner permissions and share the runtime, not hard-isolated subprocess sandboxes. No global permissions or shared runtime security boundaries are widened.

## Build and deploy

Konflux builds the runner image using the existing `.tekton/` pipelines, `dev-bot/Dockerfile.runner`, and this repository root as the build context. The image includes the pinned `dev-bot` submodule, `instance/` files, and `setup.sh` customizations.

Image builds do not deploy the workload. App-interface separately applies `deploy/template.yaml` with the selected `BOT_IMAGE`/`IMAGE_TAG` and deployment parameters. The template runs the coordinator in a SandboxWarmPool with KEDA cron scaling; shared proxy, memory service, and secrets are deployed separately. Submodule and baked `opencode-deployment.json` changes require a new image and rollout; runtime profile guidance comes from the synced config repository.

### Paired image/profile rollout

V1 rejects V2 profile fields, and the new V2 runner rejects old runtime/config fields. Treat the image and remote profile as a matching pair: merging the profile immediately affects subsequent cycles, while building an image does not deploy it. Deployment reconciliation and profile sync are independent, not an atomic cutover.

1. Preserve the prior working image digest and matching Git configuration revisions, including the gitlink, remote profile, baked deployment JSON, and deployment/image references, for paired rollback.
2. Coordinate the profile merge with image promotion. Prevent a running V1 runner from launching another cycle against the V2 profile; pause cycle launches if needed before exposing the profile change. Already-running OpenCode processes keep their loaded snapshot, but their next cycle syncs the profile again.
3. Build the onboarding image from the exact merged submodule commit above and this repository's native V2 configuration using the existing Konflux pipelines. Apply the shared [local and image validation gates](dev-bot/docs/migrations/opencode-migration.md#local-and-image-validation) to the resulting image.
4. Promote the built image through the existing app-interface image selection and reconcile the SandboxTemplate/WarmPool. Obtain a **fresh Sandbox** from that image, confirm its CLI/version and matching synced profile, then resume cycle launches. Existing GitOps parameter wiring does not establish image promotion or replace an old Sandbox.
5. Capture fresh acceptance evidence below before declaring the onboarding migration complete.

For rollback, pause new launches, restore the prior image and matching Git-backed configuration revisions together, reconcile, and obtain a fresh Sandbox before resuming. Git history preserves the prior configuration; do not combine a restored V1 image with the V2 profile or a V2 image with old configuration. No compatibility alias is provided.

### Fresh-Sandbox acceptance

Use the **REHOR-193 synthetic fixture** as the reference for separately scheduled acceptance. After coordinated rollout, capture evidence for:

- Sol → Luna → Sol delegation with wire `reasoning.effort: medium` on every model request; the five workers retain their roles, researcher/reviewer edit denials, one-level delegation limit, and assigned publication ownership.
- Jira and memory through native Code Mode, including successful nested results, distinct/deduplicated nested-operation accounting, and no wrapper budget charge against the configured `200` limit.
- Approved exact roots and descendants accessible subject to role restrictions, with external/sibling paths denied by the global policy ceiling.
- Successful cycle completion and finalized root/child evidence, provider-qualified model usage, estimated cost/provenance, and budget totals without duplicate nested charges.

Follow the shared [validation and rollback guide](dev-bot/docs/migrations/opencode-migration.md#validation-and-rollback) for the full evidence contract. Source/configuration checks and offline image tests do not establish live acceptance; fresh onboarding Sandbox acceptance remains pending.
