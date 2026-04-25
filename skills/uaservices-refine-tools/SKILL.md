---
name: uaservices-refine-tools
description: Use when you notice repeated friction in your own workflow — writing the same 6-line run_code snippet for the third time, hitting a helper that keeps needing overrides, or finding a bug that will affect every future session. The pattern: observe friction → diagnose source → edit helper → verify → commit. Also use when the user says "add a helper for this", "make this easier next time", "why do we keep having to do this manually", or "can we make the tool smarter."
---

# Refining UA Services Tools

The helper library is just Python. You can edit it. Changes persist across sessions and benefit every future agent. This is not theoretical — it's the same workflow that produced `generate_audio_seeded`, the `seed` inheritance in `generate_continuation`, and the multi-take placement docstrings.

## When to refactor vs inline

**Refactor into a helper when:**
- You've written the same run_code pattern 3+ times
- A helper's default forces you to override it on every call
- A bug will affect every future session, not just the current one
- A composite operation is logically atomic but requires 5+ lines to express

**Stay inline when:**
- It's a one-off for this session
- The logic is specific to this producer's session state
- You're not sure yet if the abstraction is right — wait for the third repetition

## The edit pattern

Four files cover almost everything:

| File | What lives there |
|---|---|
| `apps/ua_services/mcp/tools/stability.py` | Stability API wrappers, `generate_audio`, `generate_audio_seeded` |
| `apps/ua_services/mcp/tools/import_audio.py` | Placement helpers, `place_generated_clip`, `place_version`, `generate_continuation` |
| `apps/ua_services/mcp/tools/code_mode.py` | Sandbox namespace dict, `_generate_audio_sandbox`, helper docstrings for `get_runtime_helpers` |
| `apps/ua_services/mcp/tools/views.py` | Session view helpers |

To add a helper:
1. Write the function in the appropriate file
2. Add it to the sandbox namespace dict in `code_mode.py` (search for `"generate_audio": _generate_audio_sandbox` to find it)
3. Add it to the `categories` dict in `_sandbox_get_runtime_helpers` so agents discover it via `get_runtime_helpers`

## Picking up changes without a full server restart

For changes to `.py` files, reload the module inside `run_code`:

```python
import importlib
import apps.ua_services.mcp.tools.stability as _stab
importlib.reload(_stab)
my_new_fn = _stab.my_new_function
```

This works within the current session. A full server restart makes the change available as a pre-bound name in subsequent sessions.

## The dual-write rule

`stability.py`, `import_audio.py`, `code_mode.py`, and `views.py` are mirrored to the standalone `uaservices-mcp` repo. After committing a helper change to luna-vibe, sync the same file to `/tmp/uaservices-mcp-export/` and push. See luna-vibe's CLAUDE.md for the full sync workflow.

## Safety gate

Changes to the helper library persist across sessions — a bad edit affects every future agent. Apply the same judgment as any other code change:

- **Proceed autonomously:** adding a new helper, fixing a docstring, changing a default that you know is always wrong, fixing a diagnosed bug
- **Get approval first:** changing receipt shape (other tools may depend on it), removing a helper, changing apply_patch behavior, anything that touches error handling

Open a commit with a clear message. The user can revert it if it causes problems.

## Docstrings as first-class product

Layer-1 docstrings (first line before the blank line) surface in `get_runtime_helpers` at agent discovery time — they are the primary way agents learn what a helper does and when to use it. Treat them like API documentation, not code comments. When you fix a workflow confusion, fix the docstring that would have prevented it.
