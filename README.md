# Norn

Norn turns API requests into version-controlled tests your whole team can keep — authored and debugged in VS Code, run on every PR in CI. Write a request once, turn it into a tested sequence, debug it with breakpoints in the editor, and run the exact same `.norn` files from the CLI in your pipeline.

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

## Good Fit For

- backend teams validating APIs during development
- QA and automation work that needs readable test flows
- regression and smoke suites that should run the same way locally and in CI
- projects that want API requests and API tests to live next to the code
