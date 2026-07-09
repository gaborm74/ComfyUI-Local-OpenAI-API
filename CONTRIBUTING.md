# Contributing to ComfyUI-OpenAI-API

Thanks for considering a contribution! This is a small, single-purpose
ComfyUI custom node (one node, one file:
[`openai_chat_completion.py`](openai_chat_completion.py)), so the process is
intentionally lightweight.

> **Note on this repo:** this is a maintained fork of
> [bgreene2/ComfyUI-OpenAI-API](https://github.com/bgreene2/ComfyUI-OpenAI-API).
> The upstream repo appears inactive (no issues/PRs ever filed, no commits
> since 2025-10-04 as of this writing) so fixes and features are accepted
> here instead. If upstream becomes active again, functionally-scoped fixes
> from this fork are welcome to be upstreamed by anyone — no exclusivity is
> intended or implied.

## Before you start

- **Check existing issues and PRs first** to avoid duplicate work:
  [Issues](https://github.com/gaborm74/ComfyUI-OpenAI-API/issues) ·
  [Pull Requests](https://github.com/gaborm74/ComfyUI-OpenAI-API/pulls)
- **For anything beyond a trivial fix** (new inputs/outputs, behavior
  changes, new dependencies), open an issue first describing the problem
  and your proposed approach before writing code. This avoids wasted effort
  on a PR that turns out to conflict with project direction.
- **For typos, docs fixes, or obvious bugs**, just open a PR directly.

## Development setup

This node has no test suite or CI — validation is done by loading it into a
real, running ComfyUI instance and exercising the node end-to-end.

1. Fork and clone the repo into your ComfyUI's `custom_nodes/` directory:
   ```bash
   cd /path/to/ComfyUI/custom_nodes/
   git clone https://github.com/<you>/ComfyUI-OpenAI-API.git
   cd ComfyUI-OpenAI-API
   git checkout -b my-fix-branch
   ```
2. Install dependencies into whatever Python environment your ComfyUI
   instance actually uses (venv, conda, or the interpreter baked into a
   container image):
   ```bash
   pip install -r requirements.txt
   ```
3. Restart ComfyUI. Confirm the node loads without errors — check the
   ComfyUI server console output for `LLM Chat Completion` under
   `Import times for custom nodes`, and confirm it's not silently failing
   to import.

## Making changes

- **Keep the node's public surface backward-compatible where possible.**
  Prefer adding new `optional` inputs with sensible defaults over changing
  the meaning or type of an existing input — existing saved workflows
  (`.json` files) reference inputs by name, and a breaking change silently
  corrupts anyone's existing graphs.
- **New inputs need a `tooltip`** (see the `disable_thinking` input added in
  this fork for the expected style: explain *what* it does, *why* it
  exists, and roughly *what to expect* — a ComfyUI tooltip is often the only
  documentation a user ever sees for a parameter).
- **Match the existing code style** — this file uses plain functions/methods
  with no type-checking framework beyond basic type hints, `re`-based
  string processing, and the official `openai` Python SDK for all HTTP
  calls (no raw `requests`/`urllib` — keep using the SDK's `client.*`
  interface for consistency and to inherit its retry/error handling).
- **Don't crash on unexpected API responses.** ComfyUI aborts the entire
  execution graph on any unhandled exception from a node. Any code path
  that trusts a field to always be present/non-null on an LLM API response
  (e.g. `message.content`) should have a fallback — see the `result is
  None` guard added in this fork as the reference pattern.

## Testing your change

There's no automated test suite, so manual verification is required before
opening a PR. At minimum:

1. Build a minimal ComfyUI API-format workflow that exercises the node
   (a `String (multiline)` → this node → `Preview Any`/`Show Text` node is
   enough for most changes).
2. Submit it via ComfyUI's REST API (`POST /prompt`) or the browser UI, and
   confirm it completes with real output — don't just eyeball the widget in
   the editor, actually run the graph.
3. If your change affects behavior with reasoning models (`disable_thinking`,
   `strip_reasoning`, the `reasoning_tag_*` fields), test against **both** a
   reasoning-capable model (e.g. any Qwen3/Qwen3.5-family model with
   `--reasoning-parser` enabled if serving via vLLM) and a plain
   non-reasoning model, since the two code paths behave very differently.
4. If your change affects image input handling, test with **and** without
   an image connected — the `image` input is optional and the two branches
   build the request payload differently.

Include what you tested and against which backend (OpenAI, vLLM, LM Studio,
llama.cpp, Ollama, etc.) in your PR description — this substitutes for a CI
run and helps reviewers reason about coverage.

## Submitting a pull request

1. Keep PRs focused — one logical change per PR. A bugfix and an unrelated
   feature addition should be two PRs, not one.
2. Write a clear PR description: what problem does this solve, why, and how
   was it tested (see above). Link any related issue.
3. Update [`README.md`](README.md) if you're adding/changing a
   user-visible input, output, or behavior — the parameter list there is
   meant to always match `INPUT_TYPES`/`RETURN_NAMES` in the code.
4. Bump the `version` field in [`pyproject.toml`](pyproject.toml)
   (patch version for fixes, minor for new backward-compatible features) —
   this is what the [Comfy Registry](https://registry.comfy.org) uses to
   detect updates.

## Reporting bugs

Open an issue with:
- ComfyUI version (`Help > About` in the UI, or check `git log -1` in your
  ComfyUI checkout)
- The exact node inputs you used (redact API keys)
- The full error/traceback from the ComfyUI server console — not just what
  the UI shows, the console output has the real stack trace
- Which backend you're calling (OpenAI, vLLM, LM Studio, llama.cpp, etc.)
  and, if it's a self-hosted/local model, whether it's a reasoning model

## Code of conduct

Be respectful and constructive. This is a small hobby-scale project — please
keep discussion proportionate to that.
