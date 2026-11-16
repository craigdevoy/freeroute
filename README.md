# freeroute

A small MCP server that lets Claude Desktop and Claude Code send bulk, low-risk text work to free OpenRouter models.

One thread. Blocking HTTP. No SDK. About 1.8 MB binary.

## Build

    cargo build --release

The binary is at target/release/freeroute.

## Set up the key

Create the key file outside any project folder and lock it down:

    mkdir -p ~/.config/freeroute
    printf '%s' 'sk-or-v1-YOUR-KEY' > ~/.config/freeroute/key
    chmod 600 ~/.config/freeroute/key

Optional config:

    cp config.example.json ~/.config/freeroute/config.json

## Register with Claude Code

    claude mcp add --transport stdio --scope user freeroute \
      --env FREEROUTE_KEY_FILE=$HOME/.config/freeroute/key \
      --env FREEROUTE_CONFIG=$HOME/.config/freeroute/config.json \
      -- /ABSOLUTE/PATH/TO/freeroute/target/release/freeroute

The env values are paths, not the key, so nothing secret goes into ~/.claude.json.

## Register with Claude Desktop

Edit ~/Library/Application Support/Claude/claude_desktop_config.json:

    {
      "mcpServers": {
        "freeroute": {
          "command": "/ABSOLUTE/PATH/TO/freeroute/target/release/freeroute",
          "env": {
            "FREEROUTE_KEY_FILE": "/Users/YOU/.config/freeroute/key",
            "FREEROUTE_CONFIG": "/Users/YOU/.config/freeroute/config.json"
          }
        }
      }
    }

Restart Claude Desktop.

## Tools

- ask_free_model: prompt, optional system, max_tokens (default 2048, max 8192), optional model.
- list_free_models: shows the free text models in the order the server would try them.

## How it picks a model

1. Reads the live model list from OpenRouter. A model counts as free only if its id ends in :free and every price field is zero.
2. By default sends the call to openrouter/free and lets OpenRouter pick the model. It retries the router once after a 429 or a region block, because the router may pick a different model.
3. If the router fails, falls back to the top 2 ranked free models (your prefer list, then context size, then newest).
4. For large inputs (over about 32k tokens) or with `"default_route": "ranked"` in the config, tries the top 3 ranked models first and the router last.
5. A model named in the call is tried alone, with no fallback. It must be in the live free list.
6. Works inside a 75 second budget (configurable, 5 to 110). A failing ranked model is parked for a while so it is not retried at once.

Reasoning models spend max_tokens on thinking. If a reply has no answer because the limit was hit, the call stops with an error that asks for a larger max_tokens (1500 or more).

## Safety

- Free only. An explicit model must be in the live free list.
- The request body has fixed fields only: model, messages, max_tokens.
- If a reply reports a cost above zero, the server disables itself until restart.
- The model that answered must be an exact `:free` id from the live list. A served id without `:free` disables the server (and creates the kill file if one is set). A `:free` id missing from the list is discarded.
- Use an OpenRouter key with a $0 credit limit. This is the real guarantee against spend. The code checks are a second layer.
- Prompts that contain key-shaped strings are rejected.
- Output is wrapped as untrusted, cleaned of control characters, and capped at 256 KB.
- Kill switch: set FREEROUTE_KILL_FILE to a path. While that file exists, all calls are refused.
- Rate limit: 18 requests per minute, under the free cap of 20.
- Free providers may log and train on prompts. Claude must remove personal data and secrets before each call. The server does not scan for them.

## Not yet verified against the live API

- Whether openrouter/free can ever route to a paid model. In two router calls the served model kept its :free suffix.
- The tool call timeout in Claude Desktop for long answers.

## Smoke test

    printf '%s\n' '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | \
      FREEROUTE_KEY_FILE=$HOME/.config/freeroute/key ./target/release/freeroute
