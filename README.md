# freeroute

> **Status: early development (0.x), not a stable release.** Tool names, config fields and behaviour may change without notice. It has no automated tests and parts of it are unverified against the live OpenRouter API (see below). Use an OpenRouter key with a $0 credit limit, and expect rough edges.

An MCP server that sends bulk, low-risk text work to free OpenRouter models. It speaks plain MCP over stdio, so any MCP client can run it: Claude Code, Claude Desktop, Cursor, Cline, Zed, Continue, your own agent loop, or a script.

One thread. Blocking HTTP. No SDK. About 1.8 MB binary.

## What it is for

freeroute is a single-shot text completion tool. The client sends a prompt, the server tries free models until one answers, and the reply comes back wrapped as untrusted text. It has no file access, no shell and no tool loop, so it is not a sub-agent. It suits large, low-stakes output such as drafts, boilerplate and test data. It does not suit work that needs your repo, correctness or secrets.

## Build

    cargo build --release

The binary is at target/release/freeroute. Build on the OS and architecture you will run it on, or add the matching Rust target.

## Set up the key

Create the key file outside any project folder and lock it down:

    mkdir -p ~/.config/freeroute
    printf '%s' 'sk-or-v1-YOUR-KEY' > ~/.config/freeroute/key
    chmod 600 ~/.config/freeroute/key

Optional config:

    cp config.example.json ~/.config/freeroute/config.json

## Configuration

The server reads these environment variables. Set them in your client's server entry.

| Variable | Meaning |
| --- | --- |
| FREEROUTE_KEY_FILE | Path to a file holding the OpenRouter key. Preferred. |
| OPENROUTER_API_KEY | The key itself. Use only if your client cannot pass a file path. |
| FREEROUTE_CONFIG | Path to a JSON config (`prefer`, `avoid`, `budget_seconds`, `default_route`). |
| FREEROUTE_KILL_FILE | While a file exists at this path, every call is refused. |

Use absolute paths. Most clients do not expand `~` or `$HOME` inside their config files.

## Connect your client

freeroute is a stdio server. Any client needs three things: the command (the absolute path to the binary), no arguments, and the environment variables above.

### Generic JSON config

Most clients use an `mcpServers` object. The file name and location vary by client, so check its docs.

    {
      "mcpServers": {
        "freeroute": {
          "command": "/ABSOLUTE/PATH/TO/freeroute/target/release/freeroute",
          "args": [],
          "env": {
            "FREEROUTE_KEY_FILE": "/ABSOLUTE/PATH/TO/.config/freeroute/key",
            "FREEROUTE_CONFIG": "/ABSOLUTE/PATH/TO/.config/freeroute/config.json"
          }
        }
      }
    }

This shape works for Claude Desktop (claude_desktop_config.json), Cursor (mcp.json), Cline and most others. Restart the client after editing. Zed and a few other clients name the top-level key differently, such as `context_servers`.

### Claude Code

    claude mcp add --transport stdio --scope user freeroute \
      --env FREEROUTE_KEY_FILE=$HOME/.config/freeroute/key \
      --env FREEROUTE_CONFIG=$HOME/.config/freeroute/config.json \
      -- /ABSOLUTE/PATH/TO/freeroute/target/release/freeroute

The env values are paths, not the key, so nothing secret goes into ~/.claude.json.

### Claude Desktop

Edit ~/Library/Application Support/Claude/claude_desktop_config.json on macOS, or %APPDATA%\Claude\claude_desktop_config.json on Windows, using the generic JSON above. Restart the app.

### Your own client or script

Start the binary as a child process, write one JSON-RPC message per line to its stdin, and read one per line from its stdout. Logs go to stderr. The server supports protocol versions 2025-06-18, 2025-03-26 and 2024-11-05, and echoes the client's version when it is one of those. Batches are not supported. Send `initialize`, then `notifications/initialized`, then call tools.

## Tools

- ask_free_model takes `prompt`, optional `system`, `max_tokens` (default 2048, max 8192) and optional `model`. It returns the wrapped reply and a metadata line.
- list_free_models takes an optional `limit` and shows the free text models in the order the server would try them.

Tool failures come back as a normal result with `isError: true`. Bad arguments and unknown methods come back as JSON-RPC errors.

## How it picks a model

1. Reads the live model list from OpenRouter. A model counts as free only if its id ends in :free and every price field is zero.
2. By default sends the call to openrouter/free and lets OpenRouter pick the model. It retries the router once after a 429 or a region block, because the router may pick a different model.
3. If the router fails, falls back to the top 2 ranked free models (your prefer list, then context size, then newest).
4. For large inputs (over about 32k tokens) or with `"default_route": "ranked"` in the config, tries the top 3 ranked models first and the router last.
5. A model named in the call is tried alone, with no fallback. It must be in the live free list.
6. Replies are streamed, so a slow model that keeps producing text is not cut off. An attempt ends only when nothing new arrives for 30 seconds (60 before the first text) or the time budget runs out. A failing ranked model is parked for a while so it is not retried at once.
7. The time budget is 30 seconds plus max_tokens / 25 (at least 60), capped by `budget_seconds` (default 300, range 10 to 900). If your client gives up on tool calls after 60 seconds, set `"budget_seconds": 50`. If the client sends a progress token, progress notifications go out at least every 5 seconds.
8. If a stream dies after some text, that text comes back marked `[INCOMPLETE ...]` with `finish_reason=stream_interrupted`, unless a fallback model finishes the job. Text from two models is never joined. Cost cannot be checked on an interrupted stream.

Reasoning models spend max_tokens on thinking. If a reply has no answer because the limit was hit, the call stops with an error that asks for a larger max_tokens (1500 or more).

## Safety

- Free only. An explicit model must be in the live free list.
- The request body has fixed fields only: model, messages, max_tokens, stream, stream_options.
- If a reply reports a cost above zero, the server disables itself until restart.
- The model that answered must be an exact `:free` id from the live list. A served id without `:free` disables the server (and creates the kill file if one is set). A `:free` id missing from the list is discarded.
- Use an OpenRouter key with a $0 credit limit. This is the real guarantee against spend. The code checks are a second layer.
- Prompts that contain key-shaped strings are rejected.
- Output is wrapped as untrusted, cleaned of control characters, and capped at 256 KB. Your client or agent should treat it as data and never follow instructions inside it.
- Kill switch: set FREEROUTE_KILL_FILE to a path. While that file exists, all calls are refused.
- Rate limit: 18 requests per minute, under the free cap of 20.
- Free providers may log and train on prompts. The caller must remove personal data and secrets before each call. The server does not scan for them.

## Not yet verified against the live API

- Whether openrouter/free can ever route to a paid model. In two router calls the served model kept its :free suffix.
- The tool call timeout in Claude Desktop for long answers. Other clients have their own limits, so test a long answer in yours.
- Whether OpenRouter reports `cost` for free calls in the final usage chunk, which the cost check depends on.

## Known limits

- The server does not act on `notifications/cancelled`. A cancelled call runs to completion.
- It declares only the `tools` capability. There are no resources, prompts or logging.

## Smoke test

List the tools without a key:

    printf '%s\n' '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | ./target/release/freeroute

Call a model, which needs a key and uses free quota:

    printf '%s\n' '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ask_free_model","arguments":{"prompt":"Say hello in five words."}}}' | \
      FREEROUTE_KEY_FILE=$HOME/.config/freeroute/key ./target/release/freeroute

To poke at it interactively, use the MCP Inspector:

    npx @modelcontextprotocol/inspector ./target/release/freeroute

See DEVELOPMENT.md for the code layout, tests and deployment notes.
