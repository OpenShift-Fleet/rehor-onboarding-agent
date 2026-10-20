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

```bash
git submodule update --remote dev-bot
git add dev-bot
git commit -m "chore: bump dev-bot submodule"
```

## Native OpenCode configuration

- [`instance/rehor-onboarding-config/agent/instance.yaml`](instance/rehor-onboarding-config/agent/instance.yaml) selects the `onboarding` workflow, `opencode-v1` runtime, `rehor-openai` provider, and `gpt-6.1-sol` model.
- [`agent/AGENTS.md`](instance/rehor-onboarding-config/agent/AGENTS.md) supplies instance guidance alongside native core and onboarding workflow `AGENTS.md` instructions.
- [`agent/opencode.json`](instance/rehor-onboarding-config/agent/opencode.json) selects `onboarding-coordinator` as the default primary agent and limits delegation to one level. Native role files live under [`agent/.opencode/agents/`](instance/rehor-onboarding-config/agent/.opencode/agents/).
- [`opencode-deployment.json`](instance/rehor-onboarding-config/opencode-deployment.json) configures both models through the bundled `@ai-sdk/openai` Responses adapter. All roles explicitly use `reasoningEffort: medium` (ordinary/normal), including Luna implementation roles. The pinned OpenCode binary owns the bundled adapter version; the package declaration is build metadata.
- Keep the `_none` env sentinel for `install-envs.sh` (build) and `entrypoint.sh` (runtime): an empty env list currently skips env installation but makes the runtime run every preset entrypoint.

App-interface supplies `BOT_CONFIG_REPO`, `BOT_CONFIG_PATH`, `BOT_EXECUTION_ENGINE`, `REHOR_OPENCODE_DEPLOYMENT_CONFIG`, `REHOR_MODEL_PROXY_URL`, and `REHOR_MODEL_PROXY_TOKEN` through the `rehor-onboarding-agent` target in `data/services/insights/platform-frontend-ai-dev/deploy.yml`. [`deploy/template.yaml`](deploy/template.yaml) requires these parameters without defaults. The proxy token is a non-secret client placeholder; the real OpenAI API key stays on the shared proxy. The coordinator syncs the remote profile at runtime; the baked `instance/` directory is not a runtime profile fallback.

The model gateway remains `http://devbot-proxy:8450/v1`. This OpenAI-only onboarding template and its app-interface target do not declare GCP/Vertex parameters. The separately deployed shared proxy retains its GCP/Vertex configuration for other bots.

The shared proxy's `OPENAI_ALLOWED_MODELS` must include **both `gpt-6.1-sol` and `gpt-6-luna`**. Add missing models while preserving other shared allowlist entries. `VERTEX_ALLOWED_MODELS` does not authorize these OpenAI routes.

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

Roll out this migration in order:

1. Merge onboarding PR #29 first. Remote profile sync immediately exposes its profile changes to subsequent cycles.
2. Merge companion app-interface MR !214687 second to supply the new template parameters.
3. Roll out the built onboarding image containing the updated template and deployment provider models/effort.

The temporary profile/parameter mismatch until the new template/image is live is explicitly accepted, including transient cycle failures. There is no compatibility window or rollback compatibility. Already-running OpenCode processes keep their loaded configuration; restart OpenCode to load changed configuration (the next cycle starts a fresh process). Updating the shared proxy allowlist is a separate operation.
