# freeroute: developer and deployment guide

Reference for building, running, testing, deploying and changing freeroute. For what the tool does and how to register it, see [README.md](README.md).

## Requirements

- Rust 1.88 or newer (tested on 1.100). `Cargo.lock` pins crates that need 1.88+, so older toolchains fail with "rustc is not supported by the following packages".
- Update with `rustup update stable`. Check with `rustc --version`.
- An OpenRouter API key. Free models still need a key and a non-negative account balance.
- No other system dependencies. TLS is rustls via `ureq`, so no OpenSSL.

## Layout

    Cargo.toml            dependencies and release profile (size-optimised, LTO, stripped)
    Cargo.lock            committed on purpose: this is a binary, builds must be reproducible
    config.example.json   sample for FREEROUTE_CONFIG
    src/main.rs           the whole server (about 770 lines, single file)

Dependencies: `ureq` 2 (blocking HTTP), `serde`, `serde_json`. Keep it small. There is no async runtime and no MCP SDK.

## Build

    cargo build --release       # target/release/freeroute, about 1.4 MB
    cargo build                 # debug build, faster to compile, much larger

The release profile uses `opt-level = "z"`, `lto = "fat"`, `panic = "abort"` and `strip = true`. Expect a cold release build of about 30 seconds.

## Run and test by hand

The server speaks newline-delimited JSON-RPC 2.0 on stdin/stdout. stdout carries protocol messages only. Logs go to stderr.

Smoke test, no key needed for the first three calls:

    printf '%s\n' \
    '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18"}}' \
    '{"jsonrpc":"2.0","method":"notifications/initialized"}' \
    '{"jsonrpc":"2.0","id":2,"method":"tools/list"}' \
    '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"list_free_models","arguments":{"limit":5}}}' \
    | ./target/release/freeroute

Expected: an initialize result, the two tools, and a list of free models from the live OpenRouter catalog.

Live call, needs a key and uses free quota:

    printf '%s\n' \
    '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18"}}' \
    '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ask_free_model","arguments":{"prompt":"Say hello in five words."}}}' \
    | FREEROUTE_KEY_FILE=$HOME/.config/freeroute/key ./target/release/freeroute

Check the reply for the untrusted-output wrapper and the metadata line (`model=... attempts=... finish_reason=...`).

Interactive check with the MCP inspector:

    npx @modelcontextprotocol/inspector ./target/release/freeroute

## Configuration

| Variable | Purpose |
|---|---|
| `FREEROUTE_KEY_FILE` | Path to a file holding the key. Preferred. |
| `OPENROUTER_API_KEY` | Fallback if no key file is set. |
| `FREEROUTE_CONFIG` | Path to a JSON config (`prefer`, `avoid`, `budget_seconds`). Unknown fields are rejected. |
| `FREEROUTE_KILL_FILE` | While a file exists at this path, every call is refused. |

Config notes:

- `prefer` and `avoid` match as case-insensitive substrings of the model id. Free model names change often, so check `list_free_models` before relying on a `prefer` entry. A prefer list that matches nothing silently falls back to ranking by context size, then newest.
- `budget_seconds` is the ceiling for one call: clamped to 10-900, default 300. The budget actually used is the smaller of that and `30 + max_tokens / 25` seconds (at least 60). Set it to 50 for clients that time out at 60 seconds. Claude Code's tool timeout is long by default. Claude Desktop's is unverified and may be 60 seconds.
- `default_route` is `"router"` (default) or `"ranked"`. Any other value is rejected at startup. `prefer` and `avoid` apply to ranked and fallback models. For router answers, `avoid` is enforced after the fact: a served model on the avoid list is discarded and the next step is tried.

## Deploy to a new machine

1. Install Rust: `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh`
2. Clone the repo and build: `cargo build --release`
3. Put the key outside any repo:

        mkdir -p ~/.config/freeroute
        printf '%s' 'sk-or-v1-YOUR-KEY' > ~/.config/freeroute/key
        chmod 600 ~/.config/freeroute/key
        cp config.example.json ~/.config/freeroute/config.json

4. Register with Claude Code (use an absolute path to the binary):

        claude mcp add --transport stdio --scope user freeroute \
          --env FREEROUTE_KEY_FILE=$HOME/.config/freeroute/key \
          --env FREEROUTE_CONFIG=$HOME/.config/freeroute/config.json \
          -- /ABSOLUTE/PATH/TO/freeroute/target/release/freeroute

5. Or register with Claude Desktop: see the JSON block in the README, then restart Desktop.
6. Verify: run the smoke test above, then ask Claude to call `list_free_models`.

To ship a binary without a Rust toolchain on the target, build on a machine of the same OS and architecture and copy `target/release/freeroute`. The binary is self-contained. Cross-OS builds need the matching target (`rustup target add ...`).

## Updating

    git pull
    cargo build --release

Then restart Claude Desktop, or start a new Claude Code session, so the old process is replaced.

Update dependencies deliberately: `cargo update`, rebuild, rerun the smoke test, commit `Cargo.lock`.

## How the server works (for changes)

Request path in `ask`:

1. Refuse if killed, config invalid or no key.
2. Reject input over 300,000 bytes, or containing the key or a key-shaped prefix.
3. Load the model catalog (cached one hour, `ensure_catalog`). A model is free only if the id ends in `:free` and every price field is zero.
4. Build the chain: an explicit `model` alone (must be in the free list), or by default `openrouter/free` then the top 2 from `rank`. Ranked-first (top 3, then the router) when `default_route` is `"ranked"` or the input needs over 32k tokens of context. Return early if no free model has enough context.
5. For each step, `acquire` a rate-limit slot (18 per minute), then `chat`. The deadline starts before the catalog fetch. No attempt starts with under 10 s left. Before the first output an attempt keeps 15 s back for fallbacks and waits at most 60 s. Once output flows it may use all the remaining budget.

`chat` streams (`stream: true`, with `stream_options.include_usage` so cost still arrives in the last chunk) through a second ureq agent that has connect (10 s) and read (30 s) timeouts but no overall timeout, because in ureq 2 an overall `.timeout()` overrides the idle timeout. The limits are checked in the read loop:

- Socket read, 30 s: any byte resets it, including `: OPENROUTER PROCESSING` keepalives. This catches dead connections only.
- Stall clock: only a data event with non-empty `delta.content`, `delta.reasoning` or `delta.reasoning_content` resets it. 60 s before the first one, 30 s between.
- Hard deadline: the call budget.
- The first chunk's `model` is checked immediately. A non-`:free` id disables the server before any output is kept.
- A mid-stream `error` object, or `finish_reason: "error"`, counts as a dead stream.
- A stream that finished but stalled before `[DONE]` still counts as complete.
- Progress: with `params._meta.progressToken`, `notifications/progress` is written from the read loop at the first output and then at most every 5 s. `progress` is the cumulative characters received. `notifications/cancelled` is not read during a call.
6. `Outcome` decides what happens next:
   - `Ok`: wrap and return.
   - `Daily`: stop, quota gone.
   - `Stop`: stop, no other model can help (401, 402, other 403, 413, cost tripwire, reply cut off with no answer).
   - `Partial`: the stream died after some output. The longest partial is kept. A fallback only starts if the budget covers `10 + max_tokens / 50` seconds. If nothing better arrives, the partial is returned flagged, never as an error and never joined with another model's text.
   - `Skip`: park the model for a while and try the next one. A 429 or a 403 "region / not available" also gives the router one more try (`again`). A region 403 parks a ranked model for 24 hours.
7. `finish` checks the served id exactly (no `:free` stripping): no `:free` suffix sets `killed` and writes the kill file; a `:free` id outside the live list is discarded after one catalog refresh; an echoed `openrouter/free` alias is allowed with a `served_model_unverified` note. Then it sanitises the text (control, bidi and zero-width characters removed, 256 KB cap), wraps it with a random nonce, and adds a metadata line.

Invariants to keep when editing:

- The request body stays fixed: `model`, `messages`, `max_tokens`. No tools, plugins, `models` array or web search options.
- Never send a model outside the live free list, except `openrouter/free`.
- Anything on stdout must be a valid JSON-RPC message. Use `eprintln!` for logs.
- A reply with `usage.cost > 0` must set `killed`.
- Keep the single-thread, blocking design. The protocol is one request at a time.

## Known gaps

- Not yet verified against the live API: whether `openrouter/free` can route to a paid model, and the Claude Desktop tool timeout for long answers. The response `model` field kept its `:free` suffix in two router calls. Use a key with a $0 credit limit, and watch OpenRouter activity after first use.
- The `usage.cost` tripwire reads the final usage chunk. The request sets `stream_options.include_usage`, but whether OpenRouter includes `cost` for free calls in that chunk is not verified, and an interrupted stream never sends it. The request sends no `max_price`.
- The router cannot be told to pick a model with enough context, so large inputs go ranked-first.
- After a 404 the catalog is marked stale and refetched on the next call, and again on each later call if the refetch fails with no catalog, which could add a request per call on a flaky network.
- The key-shape check covers five prefixes only. Claude must strip secrets and personal data before calling. The server does not scan for them.
- There are no automated tests. The pure functions (`is_free`, `rank`, `sanitize`, `short`) are good first candidates for unit tests.

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `rustc 1.xx is not supported by the following packages` | Toolchain too old. `rustup update stable`. |
| `no API key: set FREEROUTE_KEY_FILE or OPENROUTER_API_KEY` | Env var missing in the client config, or the path is wrong. Env values must be absolute paths. |
| `cannot read config file` / `invalid config file` | Bad path or an unknown field (`deny_unknown_fields`). |
| `402` | Account credit problem. A negative balance blocks free models too. |
| `401` / `403` | Key refused. Check the key file has no stray characters. |
| `daily free quota is used up` | Resets at 00:00 UTC. |
| `freeroute is disabled` | Kill file exists, or the cost tripwire fired. Check OpenRouter activity, then restart the process. |
| `All attempts failed` | Every model in the chain failed. The error lists each reason. Parked models retry after their park time. |
| Tool does not appear in Claude | Check the absolute path, then restart Desktop or open a new Code session. `claude mcp list` shows status. |
| Nothing prints or the client hangs | Something wrote non-JSON to stdout. Run the smoke test and inspect stdout. |
