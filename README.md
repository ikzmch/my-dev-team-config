# my-dev-team-config

Shared configuration for the two MyDevTeam apps:

- **my-dev-team** (Python + LangGraph) - the autonomous batch SDLC pipeline
- **my-dev-team-vs-code** (TypeScript + Mastra) - the interactive VS Code chat agent

Both apps are capability-routed and multi-provider, and they used to maintain
two drifting model catalogues expressing the same ideas in two dialects. This
repo is the single source of truth; a sync script generates each app's native
format from it (TODO 1.1 "One shared model registry" in my-dev-team/TODO.md).

## Layout

| Path | What it is |
|---|---|
| `models.yaml` | The shared model registry - every model either app may route to |
| `scripts/sync_models.py` | Generates both apps' native registries from `models.yaml` |

Planned next (see my-dev-team/TODO.md chapter 1): shared prompt partials (1.4)
and the unified agent config format (1.3).

## Syncing

```sh
# from this repo, with my-dev-team and my-dev-team-vs-code as sibling checkouts
python scripts/sync_models.py

# non-default checkout locations
python scripts/sync_models.py --devteam ../my-dev-team --vscode ../my-dev-team-vs-code

# CI / pre-commit guard: writes nothing, exits 1 when generated files drift
python scripts/sync_models.py --check
```

The script writes:

- `my-dev-team/src/devteam/config/tools/llms.yaml`
- `my-dev-team-vs-code/src/engine/config/models/*.md` (and deletes stale files there)

**Never edit those generated files by hand** - edit `models.yaml` and re-run
the sync. Both carry a GENERATED marker. Requires Python 3.10+ and PyYAML
(`pip install pyyaml`).

## Registry schema (`models.yaml`)

Each entry under `models:`:

| Field | Required | Meaning |
|---|---|---|
| `id` | yes | Stable registry id. Becomes the vs-code model id and `<id>.md` file name |
| `label` | yes | User-facing display name (vs-code model picker) |
| `provider` | yes | Primary hosting provider (see the provider matrix below) |
| `also_via` | no | Extra providers serving the same model - the Azure mirrors. Places a copy in those `llms.yaml` sections |
| `model` | yes | Provider-specific model name: the Ollama tag or the API model id |
| `context_window` | no | Tokens. Omit when unverified - consumers then skip context-window checks |
| `thinking` | Ollama only | Whether the reasoning stream is enabled (my-dev-team's per-model Ollama flag) |
| `triage_only` | no | vs-code only: eligible for the internal triage classifier, never real work |
| `capabilities` | yes | Unified vocabulary (below), dense absolute scores in `[0, 1]` |
| `complexity_fit` | yes | Score multiplier per task complexity (`low`/`medium`/`high`), unique maximum required |
| `pricing` | yes | `{input, output}` in USD per million tokens. `null` until seeded from LiteLLM's pricing registry (TODO 3.1); `0` for local models |
| `notes` | yes | Prose note on the model's strengths - becomes the vs-code `.md` body |

Top-level, consumed only by my-dev-team's `llms.yaml`:

- `compound_providers` - pseudo-providers (like `free`) whose members reference
  registry ids; each member is emitted with an explicit `provider:` field so
  the factory dispatches to the real backend.
- `aliases` - LiteLLM model-name aliases for cost tracking, passed through.

## Unified capability vocabulary

One superset merged from both apps (TODO 1.2), used verbatim by my-dev-team's
router and translated into vs-code's 7-name dialect by the sync:

| Unified | my-dev-team | vs-code |
|---|---|---|
| `reasoning` | `reasoning` | `reasoning` |
| `code-generation` | `code-generation` | folds into `coding` (max with code-analysis) |
| `code-analysis` | `code-analysis` | folds into `coding` (max with code-generation) |
| `classification` | new, usable now | `classification` |
| `planning` | `planning` | `planning` |
| `fast-utility` | `fast-utility` | `speed` |
| `structured-output` | new, usable now | `structured-output` |
| `long-context` | new, usable now | `long-context` |

Scores sit on one dense absolute scale (the vs-code convention, kept as the
fresher of the two): 0.9x means frontier-grade, 0.5 middling, 0.3 weak. The old
my-dev-team catalogue used sparse archetype weights instead; those entries were
re-scored onto the dense scale when they moved here.

## Complexity scale

One scale, my-dev-team's `low` / `medium` / `high` naming (TODO 1.2). The
registry keeps the more expressive `complexity_fit` multipliers; the sync
derives vs-code's single `tier` as the argmax of `complexity_fit`
(`low`=`simple`, `medium`=`moderate`, `high`=`complex`), which is why the
maximum must be unique.

## Provider matrix

Each app only receives models on providers it can instantiate. Keep these
lists (in `scripts/sync_models.py`) in sync with my-dev-team's
`LLMFactory._instantiate()` and vs-code's `config/providers.ts`:

| Provider | my-dev-team | vs-code |
|---|---|---|
| ollama | yes | yes |
| groq | yes | yes |
| anthropic | yes | yes |
| openai | yes | yes |
| google | yes | yes |
| deepseek | yes | yes |
| grok (xAI) | yes | not yet (TODO 3.6) |
| mistral | yes | not yet (TODO 3.6) |
| azure-openai / azure-anthropic | yes | not yet (TODO 3.6) |
| zai | not yet (TODO 2.11) | yes |
| llamacpp | not yet (TODO 2.11) | yes |

## Adding or changing a model

1. Edit `models.yaml` (new entry, score change, retirement).
2. Run `python scripts/sync_models.py`.
3. Commit here, then commit the regenerated files in the consuming repos
   (update their model docs where the visible model set changed).

Only register models that are actually available (e.g. pulled in Ollama) -
both routers assume every registered model can run.
