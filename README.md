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
- Keep the `_none` env sentinel: an empty env list makes `entrypoint.sh` run every preset entrypoint.

[`deploy/template.yaml`](deploy/template.yaml) defaults `BOT_EXECUTION_ENGINE` to `coordinator`, `BOT_CONFIG_REPO` to this public GitHub repository, and `BOT_CONFIG_PATH` to `instance/rehor-onboarding-config`. External deployment parameters can override these defaults. The coordinator syncs the remote profile at runtime; the baked `instance/` directory is not a runtime profile fallback.

The model gateway remains `http://devbot-proxy:8450/v1`. GCP/Vertex parameters are optional, unused inputs accepted for external app-interface parameter sets; shared proxy configuration is managed separately.

The shared proxy's `OPENAI_ALLOWED_MODELS` must include **both `gpt-6.1-sol` and `gpt-6-luna`**. Add missing models while preserving other shared allowlist entries. `VERTEX_ALLOWED_MODELS` does not authorize these OpenAI routes.

### Onboarding agents

| Agent | Model | Reasoning effort | Responsibility |
|---|---|---|---|
| `onboarding-coordinator` | `rehor-openai/gpt-6.1-sol` | `medium` | One ticket/one state step per cycle; delegation and all external mutations/publication |
| `onboarding-researcher` | `rehor-openai/gpt-6.1-sol` | `medium` | Read-only repository/stack investigation |
| `onboarding-instance-scaffolder` | `rehor-openai/gpt-6-luna` | `medium` | Local Phase 1 generation, edits, and checks |
| `onboarding-konflux-implementer` | `rehor-openai/gpt-6-luna` | `medium` | Local Phase 2 Konflux changes and checks |
| `onboarding-app-interface-implementer` | `rehor-openai/gpt-6-luna` | `medium` | Local Phase 3 app-interface changes and checks |
| `onboarding-reviewer` | `rehor-openai/gpt-6.1-sol` | `medium` | Independent read-only review before publication |

The coordinator supplies scoped briefs and exact worktree paths, uses foreground implementation tasks, and resolves reviewer blockers before publishing. Workers cannot recursively delegate or directly use Jira/memory/browser MCP tools. Researchers/reviewers cannot edit or run Bash. Implementers retain inherited local edit/Bash access with direct publishing commands denied; scope and no-external-mutation instructions also apply to scripts and tests. These roles share the runtime, not hard-isolated subprocess sandboxes. No global read or external-directory permissions are widened.

## Build and deploy

Konflux builds the runner image using the existing `.tekton/` pipelines, `dev-bot/Dockerfile.runner`, and this repository root as the build context. The image includes the pinned `dev-bot` submodule, `instance/` files, and `setup.sh` customizations.

Image builds do not deploy the workload. App-interface separately applies `deploy/template.yaml` with the selected `BOT_IMAGE`/`IMAGE_TAG` and deployment parameters. The template runs the coordinator in a SandboxWarmPool with KEDA cron scaling; shared proxy, memory service, and secrets are deployed separately. Submodule and baked `opencode-deployment.json` changes require a new image and rollout; runtime profile guidance comes from the synced config repository.

For this model/agent change, build and roll out the image containing the updated deployment provider models/effort. Publish profile changes to the configured config repository for runtime sync; `agent/opencode.json`, `AGENTS.md`, and native agent files are captured for subsequent cycles. Already-running OpenCode processes keep their loaded configuration; the next cycle starts a fresh process. Updating the shared proxy allowlist is a separate operation.
