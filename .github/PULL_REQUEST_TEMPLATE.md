## What does this PR do?

<!-- One or two sentences describing the change. -->

## Why is this needed?

<!-- What problem does this solve? Link any related issue with "Fixes #N" or "Relates to #N". -->

## How was this tested?

<!-- Required — there's no CI for this repo. Describe the workflow/graph you
     tested, which backend you ran it against (OpenAI, vLLM, LM Studio,
     llama.cpp, Ollama, etc.), and whether you tested both the
     image-connected and no-image code paths if relevant. -->

## Checklist

- [ ] I tested this against a real, running ComfyUI instance (not just read the code)
- [ ] I updated `README.md` if this changes any user-visible input/output/behavior
- [ ] I bumped the `version` in `pyproject.toml` (patch for fixes, minor for new features)
- [ ] New inputs have a `tooltip` explaining what they do and why
- [ ] This change is backward-compatible with existing saved workflows (or the break is called out explicitly above)
