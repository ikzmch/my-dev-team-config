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
| `partials/*.md` | The shared prompt partials - cross-cutting prompt blocks both apps embed |
| `scripts/sync_models.py` | Generates both apps' native registries from `models.yaml` |
| `scripts/sync_partials.py` | Copies the partials into both apps, GENERATED-marked |

The unified agent config format (TODO 1.3) is converged as of 2026-07-03:
one include syntax (`{{ include <name> [if [not] <flag>] }}`), one capability
vocabulary, shared sampling keys (`temperature`/`top_p`/`top_k`) and
render-or-strip `{{tools}}`/`{{skills}}`/`{{environment}}` placeholders in
both loaders. Natural next step: graduate shared agent bodies into this repo
the way the partials did.

## Syncing

```sh
# from this repo, with my-dev-team and my-dev-team-vs-code as sibling checkouts
python scripts/sync_models.py
python scripts/sync_partials.py

# non-default checkout locations
python scripts/sync_models.py --devteam ../my-dev-team --vscode ../my-dev-team-vs-code

# CI / pre-commit guard: writes nothing, exits 1 when generated files drift
python scripts/sync_models.py --check
python scripts/sync_partials.py --check
```

The scripts write:

- `my-dev-team/src/devteam/config/tools/llms.yaml`
- `my-dev-team-vs-code/src/engine/config/models/*.md` (and deletes stale files there)
- `my-dev-team/src/devteam/config/agents/partials/*.md` (the shared subset)
- `my-dev-team-vs-code/src/engine/config/partials/*.md` (the shared subset)

**Never edit those generated files by hand** - edit `models.yaml` or
`partials/*.md` here and re-run the sync. Every generated file carries a
GENERATED marker; the partials sync only ever overwrites or deletes files
carrying that marker, so each app can keep hand-written app-local partials
(e.g. my-dev-team's `no-ask.md`) in the same directory. Requires Python 3.10+
and PyYAML (`pip install pyyaml`).

### Drift guard (pre-commit hooks)

All three repos carry a committed `.githooks/pre-commit` that runs both
scripts' `--check` mode and blocks the commit while any generated file is
stale - here so an edited `models.yaml` or partial cannot be committed before
syncing, in the consumer repos so a hand-edited generated file cannot slip in. The hooks skip silently
on machines where the sibling checkouts are absent. They resolve a Python
interpreter themselves - preferring the my-dev-team venv, since a bare
`python` on PATH may lack PyYAML - and when no interpreter with PyYAML exists
they warn and let the commit through rather than block it. Git does not
activate committed hooks by itself; enable them once per clone, in each repo:

```sh
git config core.hooksPath .githooks
```

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

## Shared prompt partials (`partials/*.md`)

Cross-cutting prompt rules both apps state (TODO 1.4): each partial is one
`.md` file whose frontmatter carries `name` (must match the file stem) and
`description`, and whose markdown body is the block. An agent prompt embeds it
with the unified `{{ include <name> }}` directive (both loaders normalise the
name - a leading path and a trailing `.md` are stripped); my-dev-team
additionally supports a conditional clause, `{{ include <name> if [not]
<setting> }}`. Bodies must not contain stray `{` or `}` - my-dev-team feeds
the resolved prompt to a LangChain template - and the sync validates this.

The six partials: `untrusted-data` (the prompt-injection guard, embedded by
every agent in both apps), `clarify-guidance` (the "ask sparingly" bar:
vs-code's triage/responder schema guidance, my-dev-team's Product Manager
when `--no-ask` is off), `scope-discipline` (strict scope + the
over-scaffolding ban), `code-style`, `faithful-reporting` and `tdd`.

## Unified capability vocabulary

One superset merged from both apps (TODO 1.2), used **verbatim by both
routers** since vs-code 0.79.0 widened its enum (the sync used to fold the two
code capabilities into `coding` and rename `fast-utility` to `speed`; vs-code
still accepts those legacy names as aliases in hand-written config like
`myDevTeam.customModels`):

`reasoning`, `code-generation`, `code-analysis`, `classification`,
`planning`, `fast-utility`, `structured-output`, `long-context`.

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
| zai | yes | yes |
| llamacpp | not yet (TODO 2.11) | yes |

## Adding or changing a model

1. Edit `models.yaml` (new entry, score change, retirement).
2. Run `python scripts/sync_models.py`.
3. Commit here, then commit the regenerated files in the consuming repos
   (update their model docs where the visible model set changed).

Only register models that are actually available (e.g. pulled in Ollama) -
both routers assume every registered model can run.
