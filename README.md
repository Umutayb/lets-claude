# lets-claude

Run **Claude Code against your own self-hosted LLM** — vLLM or any
OpenAI-compatible server that speaks the Anthropic `/v1/messages` API.

One file, no dependencies beyond `bash`, `curl`, `python3`. It probes your
endpoints, **auto-discovers the resident model and its real context window**,
wires up the `ANTHROPIC_*` environment, and launches `claude`. Works on Linux
and stock macOS (bash 3.2).

```
$ lets-claude
[lets-claude] connecting: http://localhost:8000  model: Qwen/Qwen3-32B  context: 131072
╭── Claude Code ─────────────────────────────╮
```

## Quickstart

```bash
# 1. get the script
mkdir -p ~/.local/bin
curl -fsSL https://raw.githubusercontent.com/Umutayb/lets-claude/main/lets-claude \
     -o ~/.local/bin/lets-claude
chmod +x ~/.local/bin/lets-claude

# 2. onboarding — asks for your endpoint(s), token, optional CA; saves config;
#    verifies the server; installs itself on PATH
lets-claude setup

# 3. go
lets-claude              # interactive session
lets-claude -p "hello"   # one-shot print mode (all args pass through to claude)
```

Prerequisite: the Claude CLI itself — `npm install -g @anthropic-ai/claude-cli`.

## What onboarding sets up

`lets-claude setup` walks you through everything and writes
`~/.config/lets-claude/config`:

| Question | Meaning |
|---|---|
| **Endpoints** | One or more base URLs, most-preferred first. At launch the first reachable one wins — so you can list `http://localhost:8000 https://llm.home.example` and the same script works on the server box, on your LAN, and over your VPN. |
| **Token** | Sent as the API key. vLLM without `--api-key` accepts anything — the default placeholder is fine. |
| **Root CA** | Only for `https` endpoints with a private CA (mkcert, step-ca, …). Node ignores the OS trust store, so the script exports `NODE_EXTRA_CA_CERTS` for you. |

Setup then **registers `lets-claude` as a command**:

1. asks where to install — `~/.local/bin` (per-user, default) or
   `/usr/local/bin` (system-wide, via sudo),
2. copies the script there and marks it executable,
3. if `~/.local/bin` isn't on your `PATH`, offers to append
   `export PATH="$HOME/.local/bin:$PATH"` to the right shell profile
   (`~/.zshrc`, `~/.bashrc`, or `~/.bash_profile` on macOS bash),
4. verifies with `command -v lets-claude`,
5. and finishes with a live probe that shows the model your server is
   currently serving.

After setup (and a shell restart if the PATH line was just added), typing
`lets-claude` anywhere just works.

Everything is overridable per run: `--url`, `--model`, `--insecure`,
`--verbose`, or env vars `LETS_CLAUDE_ENDPOINTS` / `LETS_CLAUDE_TOKEN` /
`LETS_CLAUDE_CA`.

## Cookbook

### Recipe 1 — serve a model with vLLM that Claude Code is happy with

Claude Code needs the Anthropic `/v1/messages` endpoint (built into recent
vLLM), tool calling, and ideally a reasoning parser so thinking renders
properly:

```bash
vllm serve Qwen/Qwen3-32B \
  --port 8000 \
  --max-model-len 131072 \
  --enable-auto-tool-choice --tool-call-parser hermes \
  --reasoning-parser qwen3
```

Pick the `--tool-call-parser` / `--reasoning-parser` matching your model
family (`vllm serve --help` lists them). Without a reasoning parser, thinking
models leak `<think>` tags into the visible output.

### Recipe 2 — why the context pin matters

Claude Code doesn't recognize self-hosted model names and assumes a **200k**
context window. If your server's `max_model_len` is smaller, long sessions
overflow the server before Claude Code auto-compacts. `lets-claude` reads
`max_model_len` from `/v1/models` at every launch and **partitions** it:
`CLAUDE_CODE_MAX_OUTPUT_TOKENS` (default 32768) is reserved for responses and
`CLAUDE_CODE_MAX_CONTEXT_TOKENS` gets the remainder. vLLM rejects any request
where `input + max_tokens` exceeds the window; with the partition, compaction
always fires before the input side can crowd out the response budget, so
those overflow 400s are impossible by construction — automatically correct
even after you swap models. Prefer longer conversations over long responses?
Launch with `CLAUDE_CODE_MAX_OUTPUT_TOKENS=16384 lets-claude` — the context
share grows to match; the two always sum to the server window.

### Recipe 3 — one config that works at home, on the LAN, and over VPN

Give `setup` an ordered endpoint list:

```
http://localhost:8000 https://llm.home.example http://server.local/vllm
```

- on the server itself → `localhost` wins (fastest, no proxy)
- on your LAN/VPN → the domain wins (put it behind nginx with
  `proxy_buffering off` so tokens stream)
- mDNS fallback for LANs without internal DNS

### Recipe 4 — private TLS with mkcert

```bash
mkcert -install && mkcert "*.home.example"      # on the server
# nginx: ssl_certificate(.key) → the generated pair
```

Copy the **public** root (`$(mkcert -CAROOT)/rootCA.pem`) to each client and
point setup's CA question at it. Never copy `rootCA-key.pem` off the server.

### Recipe 5 — thinking models

If your model thinks by default (Qwen3 family etc.), Claude Code renders the
thinking stream natively via the Anthropic API — no extra config. But avoid
small `max_tokens` in other clients hitting the same server: thinking can
consume the whole budget and return empty visible content.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `no endpoint reachable` | Server down, wrong URL, or (for domains) DNS not resolving from this network. `lets-claude --verbose` shows each probe. |
| `certificate this machine doesn't trust` | Private CA: set `CA_PATH` in `~/.config/lets-claude/config`, or `--insecure` to test. |
| Responses appear all at once, not streaming | A proxy is buffering. nginx: `proxy_buffering off; proxy_http_version 1.1;` and generous `proxy_read_timeout`. |
| `tool_choice auto` errors in logs | Server started without `--enable-auto-tool-choice --tool-call-parser <p>`. |
| Garbage `<think>` text in output | Missing `--reasoning-parser` on the server. |
| Session dies mid-way on long tasks | Context overflow — confirm the launch line shows the right `context:` value; if your server hides `max_model_len`, set `CLAUDE_CODE_MAX_CONTEXT_TOKENS` yourself. |

## License

MIT
