# lab-sandbox: design notes

Compacted from the design discussion. Not the lab itself; the outline and
the decisions behind it. Subject to change.

## What the lab covers

- **Part 0-1: dic is a thin HTTP client.**
    - dic builds the JSON body and POSTs it; conversation history lives in
      sqlite at `~/.config/fac/dic.db`, session pointers in tmpfs.
    - Show the request three ways, in increasing order of realism:
        - `--dry-run` prints URL, headers, body. Cheapest, dic-only.
        - docker + mitmproxy sees the actual wire bytes. More honest, more
          setup (cert injection into the container trust store).
        - `dic --dry-run > req; curl ... < req > resp; dic --register resp`
          reproduces a call end to end, still dic-only.
    - Students read `sqlite3 dic.db '.schema'` and `'.tables'`, and see that
      `messages` is a tree (`prev_mid`), not a list.
- **Part 2: a tool call is a JSON field.**
    - `tools[]` in the request body; a `tool_calls` entry in the reply.
      Nothing else. Demo by hand-editing a body and POSTing it with curl.
    - Then `dic --tools dic.tools.fs:*`, which is the same bytes.
    - Note the dic/llm divergence: dic has `--tools` (python functions,
      schemas read from annotations); llm does not, or does it differently.
- **Part 3: sandboxing.**
    - Tools are dangerous: arbitrary code as whatever uid dic runs under.
    - What Claude Code / Codex do by default (filesystem read is the
      common gap) vs. what a bwrap jail can do.
    - Students write two sandboxes: one wrapping docker, one wrapping bwrap.
      The docker one is slower and coarser; the bwrap one is the point.
    - **The leak.** `sandbox dic --tools=...` with `~/.config/fac` bound
      gives a bash tool read access to every session's rows, and to the
      session pointers in `$XDG_RUNTIME_DIR`. That is inter-agent
      communication, which the security model forbids.
- **Part 4: MCP, so the tool runs inside the jail.**
    - dic stays outside and holds the API key and the DB; only the MCP
      server is sandboxed. The leak closes with `sandbox` unchanged.
    - `tool.describe` -> `tools/list`: `name`, first docstring paragraph,
      `parameters`. One-to-one.
    - ~60 lines: a JSON-RPC stdio loop, `initialize` / `tools/list` /
      `tools/call`.
    - Wiring: `--tools mcp:./server.py --tools-exec sandbox-exec`, where
      `sandbox-exec` is a one-line wrapper that `exec sandbox -- "$@"`.

## Tradeoffs

- **mitmproxy vs `--dry-run`/`--register`.** Proxy shows real bytes and
  exercises docker and cert management; `--dry-run` is half the setup and
  works for one model at a time. The proxy detour is worth its complexity
  only if docker is being taught anyway.
- **Where the boundary is.** Three placements, all defensible:
    - *Sandbox dic itself.* Simple, but the DB is inside the jail, so a
      tool can read across sessions. Fixable by projecting the DB only, at
      the cost of merge races and more machinery than MCP.
    - *`--tool-exec`.* Sandbox the tool process, not dic. Requires dic to
      speak a protocol to its own tools, which is what MCP already is.
    - *MCP over stdio.* dic is unchanged in shape; the protocol is the
      refactor. Adds one part to the lab rather than a new layer to dic.
- **Who applies the sandbox.** `--tools-exec` lets the agent sandbox its
  own tools; running the server under `sandbox-exec &` and pointing dic at
  it keeps the privilege with the operator. More honest architecturally,
  less reproducible for a lab.
- **`--unshare-net` vs remote servers.** Stdio MCP needs no network. A
  server that talks to Postgres over a unix socket needs that socket
  bound; one over TCP needs `--share-net`, which reopens exfil. The jail
  forces the network decision into the open, which is the lesson.
- **MCP vs an ad-hoc protocol.** MCP is not shorter in lines; it is shorter
  in *decisions*. Version pinning and `isError` mapping are the two places
  students will spend time.

## Partner interaction

- **Remote MCP over port forward.** Partner A runs a containerized
  Postgres plus an MCP server whose one tool is `sql` (`docker exec ...
  psql`); Partner B port-forwards, points dic at it, and must answer a
  question only A's data answers. Swap.
    - Lands docker, sql, networking, MCP, and sandbox in one artifact.
    - Makes the trust boundary concrete: whose jail protects whose data.
- **Red/blue variant.** A's server is deliberately hostile (exfil attempt,
  or a write into B's DB through a path B forgot to jail). B sandboxes the
  *server* before connecting; A verifies the attack failed.
    - Sharp lesson: the boundary is where the process runs, not where the
      call is made. Port forwarding sandboxes nothing on the remote side.
- **Patchfile exchange** (reuse of lab-coding-agent). Each writes a
  `dic.tools.X` module, ships it as a git patch, partner applies and runs
  `dic --tools=dic.tools.X:*`.

## Open questions

- Length. MCP replaces `--tool-exec` rather than adding to it, so the
  slope is: dry-run -> tools-as-JSON -> sandbox -> MCP. Still tight, but
  the partner part in Part 4 is where it could be cut if needed.
- Nondeterminism. Pin the model, `-o temperature=0`, ship reference
  patches. Do not let a bad reply cascade into the next part.
