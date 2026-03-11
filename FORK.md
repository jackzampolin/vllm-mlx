# jackzampolin/vllm-mlx Fork

Amygdala's maintained fork of [waybarrios/vllm-mlx](https://github.com/waybarrios/vllm-mlx) with cherry-picked community PRs for Qwen3.5 MoE support.

## Why a Fork

Upstream vllm-mlx has open PRs that fix critical issues for Qwen3.5-397B-A17B (our primary model) but the maintainer hasn't merged them. This fork integrates tested, high-impact PRs so we can deploy them now.

## Active Branch

**`amygdala/qwen35-integration`** — production branch, deployed to Studios via `infra/services/vllm/deploy-vllm`.

Built on upstream `main` (v0.2.6) with these cherry-picks:

| PR | What | Impact |
|----|------|--------|
| [#144](https://github.com/waybarrios/vllm-mlx/pull/144) | Fix 3D KV tensors in prefix cache for Qwen3.5 | Prefix cache was silently disabled on MoE models. 63% faster on cache hits. |
| [#127](https://github.com/waybarrios/vllm-mlx/pull/127) | Qwen3.5 text-only loading + dynamic memory threshold | `strict=False` fallback, fixes reasoning+tool streaming, dynamic memory pressure |
| [#132](https://github.com/waybarrios/vllm-mlx/pull/132) | Anthropic streaming scrubber + thinking blocks | Stops `<think>`/`<tool_call>` markup leaking into responses |
| [#148](https://github.com/waybarrios/vllm-mlx/pull/148) | Reasoning + tool call parser bridge | Fixes tool calls emitted inside reasoning blocks |

## Syncing with Upstream

```bash
cd /Users/johnzampolin/go/src/github.com/jackzampolin/vllm-mlx
git fetch upstream
```

### When upstream merges one of our PRs

If upstream merges (say) PR #144, rebase the integration branch to drop the now-redundant commit:

```bash
git checkout amygdala/qwen35-integration
git rebase -r upstream/main
# Resolve any conflicts (our cherry-pick vs their merge)
git push --force-with-lease origin amygdala/qwen35-integration
```

Then redeploy: `./infra/services/vllm/deploy-vllm studio`

### When upstream releases a new version

```bash
git fetch upstream
git log --oneline upstream/main..amygdala/qwen35-integration  # see what we still carry
```

If upstream has caught up on all our cherry-picks, switch back to PyPI: set `VLLM_SOURCE=pypi` in all instance envs and remove the fork config from `common.sh`.

### Cherry-picking a new PR

```bash
# Fetch the PR
git fetch upstream pull/<NUMBER>/head:pr/<NUMBER>

# Apply it
git checkout amygdala/qwen35-integration
git cherry-pick pr/<NUMBER>

# If conflicts, resolve and commit
# Then push and redeploy
git push origin amygdala/qwen35-integration
```

Update the table above when adding new cherry-picks.

## PRs Worth Watching

These are open upstream PRs that may be worth pulling in later:

| PR | What | Status |
|----|------|--------|
| [#140](https://github.com/waybarrios/vllm-mlx/pull/140) | Qwen3.5 full MLLM/VLM support (vision) | Large, needs testing |
| [#125](https://github.com/waybarrios/vllm-mlx/pull/125) | `--served-model-name` CLI param | Small, useful for aliases |
| [#126](https://github.com/waybarrios/vllm-mlx/pull/126) | Fix base64 image cache collision | Correctness fix for VLM |

## Testing Before Deploy

```bash
# Run unit tests (no model required)
python3 -m pytest tests/ -x -m "not slow" --tb=short

# Verify no import errors
python3 -c "from vllm_mlx.server import app; print('ok')"
```

## Deploying

Deployment uses the standard amygdala service scripts. The fork is installed via `uv tool install` from git:

```bash
# Deploy to studio-1 only (fork)
./infra/services/vllm/deploy-vllm amygdala-1

# Check status
./infra/services/vllm/status-vllm amygdala-1

# Deploy to all studios
./infra/services/vllm/deploy-vllm studio
```

See `infra/services/vllm/common.sh` for `VLLM_FORK_REPO` and `VLLM_FORK_BRANCH` configuration.
