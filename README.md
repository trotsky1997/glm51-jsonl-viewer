# GLM51 JSONL Viewer

Browser viewer for ShareGPT/GLM51 SFT JSONL and ATIF/Pi trajectory files.

Demo: https://trotsky1997.github.io/glm51-jsonl-viewer/

## What It Does

This is a static HTML viewer. It does not upload your data. You open a local
`.jsonl`, `.json`, `.atif`, or `.txt` file in the browser, and the page renders
records on demand.

It is built for inspecting agent SFT traces:

- GLM-5.1 training records with `messages`, `tools`, `chat_template_kwargs`, and
  `meta`
- ATIF/Pi trajectories with `steps`, `tool_calls`, observations, reasoning
  content, and final metrics
- Tool calls and tool responses colocated in the conversation view
- Markdown, code blocks, inline JSON, bash commands, read/write/edit operations,
  and GitHub-style diffs
- Token reports using `js-tiktoken` with `cl100k_base`

## Usage

Open the demo page or open `index.html` directly in a browser.

1. Click the file picker in the top-left.
2. Select a local JSONL/JSON/ATIF file.
3. Use search and filters to narrow records.
4. Click a record in the left list to render its detail.

Loading a file only builds the left-side index. The viewer does not render the
first record automatically, so large traces do not trigger heavy rendering until
you click a row.

## Supported Input

GLM51 SFT JSONL:

```json
{
  "messages": [
    {"role": "user", "content": "task"},
    {
      "role": "assistant",
      "content": "answer",
      "reasoning_content": "optional reasoning",
      "tool_calls": []
    }
  ],
  "tools": [],
  "chat_template_kwargs": {"enable_thinking": true},
  "meta": {"status": "pass"}
}
```

ATIF/Pi trajectory:

```json
{
  "schema_version": "atif-...",
  "session_id": "session",
  "agent": {"model_name": "model"},
  "steps": [
    {
      "step_id": 1,
      "source": "agent",
      "message": "response",
      "reasoning_content": "optional reasoning",
      "tool_calls": []
    }
  ],
  "final_metrics": {
    "total_prompt_tokens": 123,
    "total_completion_tokens": 45,
    "total_cached_tokens": 67
  }
}
```

JSONL rows may also wrap an ATIF object under `atif`.

## How It Works

For large JSONL files, the viewer indexes byte offsets instead of loading every
full record into memory. During indexing it parses each line once, keeps a small
summary for the left list, and stores `start/end` offsets for lazy detail reads.

When you click a record, the viewer reads only that JSONL slice with
`file.slice(start, end)`, parses it, normalizes it, and renders the conversation.
Recently opened records are kept in a small in-memory cache.

Rendering is DOM-based:

- Markdown is parsed with `marked`
- Inline rich text uses `pretext` when available
- Code highlighting uses `highlight.js`
- Tool calls are normalized into OpenAI-style function calls
- Tool responses are paired with nearby tool calls by `tool_call_id`
- `read`, `bash`, `write`, and `edit` get specialized renderers

The left list is virtualized. Only visible rows are mounted in the DOM, so
scrolling remains usable on large files.

## Token Report

The token report uses `js-tiktoken` with `cl100k_base`.

It reports visible trace tokens by category:

- `system`
- `user`
- `assistant reasoning`
- `assistant answer`
- `tool call`
- `tool response`
- `tool schema`

If the record has API usage in `meta.final_metrics`, `meta.usage`,
`meta.token_usage`, or `meta.tokens`, the viewer also shows API prompt,
completion, cached, and total tokens.

The visible trace count is an estimate for GLM-5.1 training length because it
uses OpenAI-compatible `cl100k_base`, not the GLM tokenizer. It is useful for
inspection and relative comparison. For exact GLM training length, compute with
the training tokenizer and template offline.

## CDN And Cache

The app is intentionally static. Runtime dependencies are loaded from pinned CDN
URLs:

- `@chenglou/pretext@0.0.7`
- `marked@18.0.4`
- `highlight.js@11.11.1`
- `js-tiktoken@1.0.21`

The page preconnects and module-preloads these dependencies. On GitHub Pages it
also registers `sw.js`, which caches the app shell and CDN modules. First load
still needs network access; later loads can reuse the browser cache. User files
are never cached by the service worker.

## Local Development

The viewer is a static page:

```bash
python3 -m http.server 8000
```

Then open:

```text
http://localhost:8000/
```

You can also open `index.html` directly, although service-worker caching only
works over HTTPS or localhost.
