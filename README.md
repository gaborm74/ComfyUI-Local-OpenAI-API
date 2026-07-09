# ComfyUI-Local-OpenAI-API

> **This is a fork of [bgreene2/ComfyUI-OpenAI-API](https://github.com/bgreene2/ComfyUI-OpenAI-API)**,
> renamed to **ComfyUI-Local-OpenAI-API** and maintained here because the
> upstream repo appears inactive (0 issues/PRs ever filed, last commit
> 2025-10-04, no CONTRIBUTING guide — checked 2026-07-09 before forking).
> This fork adds one feature and one bugfix needed for reasoning-model
> support; see **[Changes in this fork](#changes-in-this-fork)** below. All
> original functionality and credit is unchanged.

This is a custom node that calls an LLM and outputs the resulting text.

![Node screenshot](assets/node_screenshot.png)

## Features

- Supports any OpenAI-compatible endpoint (including those you host locally, such as with llama.cpp, LMStudio, vLLM, or other programs)
- Extracts reasoning content from the output of thinking models
- Supports either text-only input or text + image input for VLMs
- **(this fork)** Optional per-request `disable_thinking` toggle for always-thinking reasoning models (see below)

## Installation

1. Navigate to your custom_nodes folder
2. Clone this repo: `git clone https://github.com/gaborm74/ComfyUI-Local-OpenAI-API.git`
3. Change to the directory `cd ComfyUI-Local-OpenAI-API`
4. Assuming the correct Python environemnt is loaded, install dependencies `pip install -r requirements.txt`
5. Restart ComfyUI

## Usage Guide

The node has the following inputs:

- text_prompt - Your prompt to send to the LLM
- image (optional) - An image to send to the LLM

The node has the following parameters:

- system_prompt - If given, this will be sent to the LLM as the system prompt
- pre_prompt - If given, this will be added before the text_prompt to construct the actual user message sent to the LLM
- endpoint - The hostname or ip address where the API is located. You may need to include "/v1" at the end.
- model - The name of the AI model to request at the API
- sleep - Amount of time, in seconds, to delay after calling the LLM but before returning the output
- use_temperature - Whether to send a temperature
- temperature - The temperature to send
- use_seed - Whether to send an RNG seed. If this is not specified, the default behavior of the endpoint will be used, which is probably to use a random seed.
- seed - The seed to send
- control after generate - This is attached to the 'seed' parameter
- use_max_tokens - Whether to send max_tokens
- max_tokens - The value for max_tokens to send
- strip_reasoning - Whether to post-process the returned text to split out the reasoning and non-reasoning content
- reasoning_tag_open - The string that starts reasoning content
- reasoning_tag_close - The string that ends reasoning content
- **disable_thinking** *(this fork, default `False`)* - When enabled, sends `chat_template_kwargs: {"enable_thinking": false}` as `extra_body` on the chat-completion request. See [Changes in this fork](#changes-in-this-fork) for why this exists.

The node has the following outputs:

- Text - The text content after calling the LLM
- Reasoning - The reasoning content after calling the LLM

For basic usage, leave everything default except for `endpoint` and `model`. Connect a `String (multiline)` node to the `text_prompt` input. Connect the `Text` output to two nodes: a `Preview Any` node, and the text input of your text encoder node (might be named like `CLIP Text Encode` or similar).

![Basic usage example (before)](assets/basic_usage_example_before.png)
![Basic usage example (after)](assets/basic_usage_example_after.png)

See [this example workflow](workflows/flux_dev_example_openai_api.json).

## Recommended Usage

This node is ideal for prompt enhancement, i.e. you write a basic prompt and the LLM jazzes it up to give more detailed or varied images.

This node is a good option if you are using an API hosted on a different machine. The `sleep` parameter is provided to facilitate hosting the API on the same machine. You can run an API server that allows setting a ttl of 1 second (e.g. [llama-swap](https://github.com/mostlygeek/llama-swap), Ollama, etc), then set `sleep` to 2 seconds. The node will call the LLM server, the server will process the request and then unload the model, then the workflow can proceed.

## Recommended Models

This node was tested with [PromptEnhancer-32B](https://huggingface.co/PromptEnhancer/PromptEnhancer-32B).

It has also been validated (in this fork) against a locally-served **vLLM**
endpoint running a Qwen3.5-architecture reasoning model
(`YuYu1015/YuYu1015-Ornith-1.0-9B-abliterated`, served as
`ornith1-9b-abliterated`) for text-to-image prompt enhancement, replacing an
Ollama-based prompt-expansion sidecar in a ComfyUI deployment.

## Changes in this fork

**Problem encountered:** always-thinking reasoning models (Qwen3 /
Qwen3.5-family, served via vLLM with `--reasoning-parser qwen3`) open every
response with an internal `<think>...</think>` pass before emitting the
actual answer. Against a real deployment this caused two distinct failures:

1. **Crash.** With a modest `max_tokens` budget (e.g. 200 — reasonable for a
   short prompt-enhancement task on a non-reasoning model), the model spent
   the entire budget on reasoning and hit `finish_reason: "length"` before
   ever emitting real content. vLLM then returns `message.content: null`.
   The node's reasoning-stripper (`re.findall(pattern, result, ...)`) calls
   a regex directly on `result` with no `None` check, raising
   `TypeError: expected string or bytes-like object, got 'NoneType'` and
   failing the whole ComfyUI graph.
2. **Latency.** Raising `max_tokens` high enough to avoid the crash (e.g.
   1500) fixes the exception, but ~800 of those tokens get spent on
   reasoning before the real answer appears — a ~72 second round trip for a
   one-paragraph prompt enhancement on a 9B model on local hardware
   (DGX Spark / GB10).

**Fix — a `disable_thinking` toggle (default `False`, opt-in, fully
backward-compatible):**

- When enabled, the node passes `extra_body={"chat_template_kwargs":
  {"enable_thinking": False}}` to `client.chat.completions.create(...)`.
  This is the standard mechanism Qwen3/Qwen3.5 chat templates expose for
  suppressing the `<think>` block entirely at generation time, rather than
  generating it and stripping it after the fact.
- Measured effect on the model above: **~72s → ~7s** (roughly 10x) for the
  same prompt, temperature, and seed, with comparable output quality.
- Endpoints/models that don't recognize `chat_template_kwargs` simply ignore
  the extra field — this is a silent no-op for non-reasoning models or
  providers that don't support it, so it's safe to leave the checkbox
  visible without affecting existing workflows.
- Also added a defensive `if result is None: result = ""` guard right after
  the API call, independent of the toggle above — so a `None` content (from
  hitting `finish_reason: "length"` for *any* reason, not just this specific
  scenario) degrades to an empty string instead of crashing the node.

**Why a fork instead of a PR upstream:** checked
[bgreene2/ComfyUI-OpenAI-API](https://github.com/bgreene2/ComfyUI-OpenAI-API)
on 2026-07-09 for an existing issue/PR covering this scenario (none found)
and for signs of active maintenance (0 issues or PRs ever filed, last commit
2025-10-04, 4 commits total, no CONTRIBUTING guide, single-star personal
utility repo, the only fork is an automated Comfy-Registry mirror bot with
identical commit history to the origin). Given no evidence of an active
maintainer to review a contribution, maintaining a patched fork was more
practical than opening a PR that would likely go unreviewed.

## Credit

- Originally based on https://github.com/tianyuw/ComfyUI-LLM-API
- This fork's `disable_thinking` parameter and `None`-content guard: gaborm74, 2026-07-09

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for setup, coding style, testing
expectations, and the PR process.
