# Norn

Norn keeps API tests and database checks in repeatable, version-controlled files your whole team can trust. Author, inspect, and debug them in VS Code, then run the same files from the CLI and CI.

### Simple API Requests

![Send requests and run sequences](https://firebasestorage.googleapis.com/v0/b/norn-1a99c.firebasestorage.app/o/simple.gif?alt=media&token=6625ec07-041c-4900-a23e-f09df0e9bf99)

### Chain and Debug API Requests

![Debug sequences in VS Code](https://firebasestorage.googleapis.com/v0/b/norn-1a99c.firebasestorage.app/o/debug.gif?alt=media&token=2b62c4d4-44d4-403b-ad03-2c0a39b41260)

## Why Norn

Most API tools split the work across too many places: one app for sending requests, another for test logic, shell scripts for CI, and a pile of copied values between them. Norn keeps that work in plain text files inside VS Code.

That means you can:

- send single HTTP requests without leaving the editor
- build reusable sequences with variables, captured values, assertions, waits, retries, and branching
- debug those sequences with breakpoints and step-through execution
- run the exact same files from the CLI for smoke tests, regression suites, and pipelines

## What You Get

- `.norn` files for requests, sequences, and tests
- `.nornenv` files for environments and secrets
- `.nornapi` files for reusable endpoint definitions
- `.nornsql` files for database queries and commands
- `.nornagent` files for contract-checked AI agents and model-directed delegation
- syntax highlighting, IntelliSense, and diagnostics
- response inspection, JSON diffing, and click-to-generate assertions
- tagged and parameterized test execution in VS Code and the CLI

## Example

```norn
var baseUrl = https://api.example.com

sequence Checkout
    POST {{baseUrl}}/auth/login
    Content-Type: application/json
    {
        "username": "demo",
        "password": "secret"
    }

    var token = $1.body.accessToken

    GET {{baseUrl}}/orders
    Authorization: Bearer {{token}}

    assert $2.status == 200
end sequence
```

This is the core idea: one file can hold the request, the flow, the captured data, and the assertion. When the flow grows, you still stay in text, version control, and normal code review.

## In VS Code

Use Norn to:

- send a request directly from a `.norn` file
- run a whole sequence from the editor
- debug a sequence with breakpoints
- run test sequences from the Testing view

## In The CLI

The CLI uses the same execution model as the extension, so local runs and CI runs stay aligned.

```bash
npm install -g norn-cli
norn ./tests/smoke.norn -e dev
```

## `.nornenv` Templates And Extends

Use `[template:name]` sections for reusable environment building blocks, then compose selectable `[env:name]` sections with `extends`. Only templates can be extended. Templates are not selectable from the VS Code environment picker or CLI; only `[env:...]` names can be used with `-e`.

```nornenv
var timeout = 30000

[template:prod]
var baseUrl = https://api.example.com
secret apiKey = prod-key-789

[template:uk]
var dbHost = db.uk.example.com
var bucket = data-uk

[env:prod_uk extends prod, uk]
var failoverHost = api-failover.uk.example.com
```

Resolution order is `common <- template1 <- template2 <- self`, so later templates win on collisions and the env section itself wins over everything. The VS Code editor also shows Activate CodeLens actions, inherited-variable peeks, hover resolution chains, and inlay hints for `{{name}}` / `{{$env.name}}` references.

## Inlay Hints Across Every File Type

Every `{{...}}` reference in any Norn file shows its resolved value as a gray inline hint under the active env, with the same masking-for-secrets rules as `.nornenv`. The three scopes available in `.norn` resolve in precedence order:

1. **Sequence-local** `var` declarations inside the current `test sequence ... end sequence` block
2. **File-level** `var` declarations at the top of the file (above any sequence)
3. **Active env** effective vars (`common` ← ancestor templates ← self)

`{{$env.name}}` always reads from scope 3, skipping local/file vars. Runtime values (`$1.body.id`, request captures, `run X()` returns) render no hint inline; hover narrates them with source line. `.nornapi` and `.nornsql` use env scope only; `.nornsql` additionally shows the resolved connection string after `connection NAME`.

## Deterministic MCP Tools

Norn can call MCP tools from sequences without leaving the `.norn` runtime. MCP sessions are deterministic and shared across the full sequence run, so nested sequences reuse the same connection for the same resolved server alias.

Create a `norn.config.json` in the root of your project:

```json
{
    "version": 1,
    "mcp": {
        "servers": {
            "localTools": {
                "transport": "stdio",
                "command": ["node", "./tools/mcp-server.js"]
            },
            "remoteTools": {
                "transport": "http",
                "url": "https://mcp.example.com/mcp",
                "headers": {
                    "Authorization": "Bearer {{$env.mcpToken}}"
                },
                "timeoutMs": 5000
            }
        }
    }
}
```

Use MCP tools directly inside sequences:

```norn
sequence ToolFlow
    var tools = run mcp list localTools
    var result = run mcp call localTools summarize_text(text: "hello world", format: "short")

    assert tools[0].name exists
    assert result.structuredContent.summary exists
end sequence
```

Behavior:

- `run mcp list <alias>` returns the full tool list and drains paginated `nextCursor` responses automatically.
- `run mcp call <alias> <tool>(...)` supports named arguments or positional arguments bound in tool-schema order, and returns a deterministic result envelope with `content`, `structuredContent`, `isError`, `text`, `server`, and `tool`.
- Tool `structuredContent` is validated against the MCP tool's advertised `outputSchema` when present.
- Sessions are closed automatically when the outermost sequence finishes or fails.

## Contract-Checked Agent Graphs

Define agents in an imported `.nornagent` sidecar. An agent can grant another
agent through `agents`; the callee then appears to the calling model as an
ordinary tool. Its `accepts` schema constrains and validates the generated tool
input, and its `returns` schema validates the result before it travels back up
the graph.

```nornagent
model Workbench: openai/gpt-4o

agent DomainExpert
    model Workbench
    describe "Call for domain questions and include the ticket context."
    accepts contracts/domain-question.schema.json
    returns contracts/domain-answer.schema.json
    system "Answer only the supplied domain question."
end agent

agent TicketRouter
    model Workbench
    agents DomainExpert
    system "Consult the domain expert when the ticket needs it."
end agent
```

A prompt that outgrows its sidecar can live in its own file instead. `describe`
and `system` both accept `file <path>`, resolved relative to the `.nornagent`
file just like an import:

```nornagent
agent TicketRouter
    model Workbench
    agents DomainExpert
    system file prompts/ticket-router.md
end agent
```

The file's text is the prompt verbatim — no escaping, so quotes and backslashes
stay as written — and `{{...}}` references in it resolve exactly as they do
inline. A missing or empty prompt file is a parse error on the directive line.

Run the graph from an ordinary sequence; the same nested contract and retry
trace appears in VS Code and the CLI:

```norn
import "./agents.nornagent"

test sequence RouteTicket
    var ticket = run readJson "./ticket.json"
    var verdict = run TicketRouter ticket
    assert verdict.text exists
end sequence
```

Malformed handoffs are returned to the calling model as field-level tool errors
so it can correct them. The default guardrails are depth 5, 25 total agent
invocations, and a two-failed-attempt contract cap. Configure them globally,
per provider, or on an individual agent (most specific wins):

```json
{
    "version": 1,
    "agents": {
        "max_tokens": 16000,
        "max_depth": 5,
        "max_invocations": 25,
        "contract_retries": 2,
        "recording": { "enabled": true },
        "providers": {
            "local": { "max_tokens": 4096 }
        }
    }
}
```

`max_tokens` is an output ceiling, not prepaid usage: raising it does not spend
tokens by itself. Depth, invocation, and retry limits can permit additional
model calls, so tighten those three when controlling run cost. See the runnable
[`demos/agent-workbench`](./demos/agent-workbench) example for delegation and a
visible contract correction.

Every sequence run that reaches an agent is recorded under
`.norn-cache/runs/` by default, with a rolling cap of 20 files. Recordings hold
the resolved provider request, response, tool transcript, contracts,
conversation state, and ordered hop events. Values declared as secrets in
`.nornenv` are written as stable named placeholders and restored from the
selected environment when replay starts; a missing value stops replay before a
provider or tool can run. Set `agents.recording.enabled` to `false` to disable
automatic recording.

Replay a recording without model calls, MCP calls, or other sequence side
effects using the local CLI:

```bash
node ./dist/cli.js replay .norn-cache/runs/<recording>.json
node ./dist/cli.js replay .norn-cache/runs/<recording>.json --json
node ./dist/cli.js replay .norn-cache/runs/<recording>.json --from CriteriaComparer
```

Pure replay reproduces the stored trace and exits non-zero for failed hops or
contracts, so a deliberately copied recording can be used as a no-key CI
fixture. `--from` accepts a unique agent name or canonical path such as
`TicketRouter[1]/CriteriaComparer[1]`; the prefix stays replayed and that hop
onward runs live against the current graph.

**Norn: Show Agent Graph**, or the CodeLens on any agent block, draws the graph
for a `.nornagent` file: who calls whom, the `accepts`/`returns` contract on each
boundary, and how each one turned out. Hover an agent or a boundary for the full
detail — models, timings, usage, prompts, payloads, contract issues.

The graph's toolbar also lists the project's recorded runs. Pick one to see how
it finished, or press **Play run** to watch it unfold hop by hop on the canvas.
Playback replays the recording's own events, so nothing re-executes and no model
is called; press **Stop** to jump to the end. Recording is on by default and
keeps the last 20 runs under `.norn-cache/runs/`.

To step a recording in VS Code, use the existing `norn` debugger and add the
artifact to a launch configuration:

```json
{
    "type": "norn",
    "request": "launch",
    "name": "Replay RouteTicket",
    "file": "${workspaceFolder}/route-ticket.norn",
    "sequence": "RouteTicket",
    "recording": "${workspaceFolder}/fixtures/route-ticket-run.json",
    "stopOnEntry": true
}
```

Agent calls appear as nested frames with Request, Response, Contracts, and
Conversation scopes. Normal step controls navigate the recorded hop timeline;
**Norn Debug: Run to Next Contract Failure** stops on retry violations as well
as final contract failures. With Agent Graph open, the paused invocation is
highlighted and selecting a node moves the replay cursor to that invocation.

## Good Fit For

- backend teams validating APIs during development
- QA and automation work that needs readable test flows
- regression and smoke suites that should run the same way locally and in CI
- projects that want API requests and API tests to live next to the code

Diff preview test line.

ping
