# Norn - REST Client

A powerful REST client extension for VS Code with sequences, variables, script execution, and CLI support for CI/CD pipelines.

![VS Code Version](https://img.shields.io/badge/VS%20Code-1.108.1+-blue)
![License](https://img.shields.io/badge/license-MIT-green)

## Features

- **HTTP Requests**: Send GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS requests from `.norn` files
- **Variables**: Define and reference variables with `var name = value` and `{{name}}`
- **Sequences**: Chain multiple requests with response capture using `$N.path`
- **Script Execution**: Run bash, PowerShell, or JavaScript scripts within sequences
- **Print Statements**: Debug and log messages during sequence execution
- **Cookie Support**: Automatic cookie jar with persistence across requests
- **Syntax Highlighting**: Full syntax highlighting for requests, headers, JSON bodies
- **IntelliSense**: Autocomplete for HTTP methods, headers, variables, and keywords
- **Diagnostics**: Error highlighting for undefined variables
- **CLI**: Run requests from terminal for CI/CD automation

## Quick Start

1. Create a file with `.norn` extension
2. Write your HTTP request
3. Click "Send Request" above the request line

## Usage

### Basic Request

```http
GET https://api.example.com/users
Authorization: Bearer my-token
```

### Variables

```http
var baseUrl = https://api.example.com
var token = my-secret-token

GET {{baseUrl}}/users
Authorization: Bearer {{token}}
```

### Request with Body

```http
POST https://api.example.com/users
Content-Type: application/json
{
    "name": "John Doe",
    "email": "john@example.com"
}
```

### Sequences (Chained Requests)

Chain multiple requests and capture response data:

```http
sequence AuthFlow
    POST https://api.example.com/login
    Content-Type: application/json

    {"username": "admin", "password": "secret"}

    var token = $1.accessToken

    GET https://api.example.com/profile
    Authorization: Bearer {{token}}
end sequence
```

Click "▶ Run Sequence" above the `sequence` line to execute all requests in order.

### Script Execution

Run scripts within sequences for setup, data generation, or validation:

```http
sequence TestWithScripts
    # Run a setup script
    run bash ./scripts/seed-db.sh

    # Generate a signature and capture output
    var signature = run js ./scripts/sign.js {{payload}}

    # Use the signature in a request
    POST https://api.example.com/verify
    X-Signature: {{signature}}
end sequence
```

Scripts receive variables as environment variables with `NORN_` prefix (e.g., `NORN_TOKEN`).

### Print Statements

Add debug output to your sequences:

```http
sequence DebugFlow
    print Starting authentication...
    
    POST https://api.example.com/login
    Content-Type: application/json
    {"user": "admin"}
    
    var token = $1.token
    print Token received | Value: {{token}}
end sequence
```

Use `print Title | Body content` for expandable messages in the result view.

## CLI Usage

Run requests from the command line for CI/CD pipelines:

```bash
# Run a specific sequence
npx norn api-tests.norn --sequence AuthFlow

# JSON output for CI/CD
npx norn api-tests.norn --sequence AuthFlow --json

# Verbose output
npx norn api-tests.norn -v

# Show help
npx norn --help
```

### CLI Options

| Option | Description |
|--------|-------------|
| `-s, --sequence <name>` | Run a specific sequence by name |
| `-j, --json` | Output results as JSON (for CI/CD) |
| `-v, --verbose` | Show detailed output |
| `--no-fail` | Don't exit with error code on failed requests |
| `-h, --help` | Show help message |

### CI/CD Example (GitHub Actions)

```yaml
jobs:
  api-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
      
      - name: Run API Tests
        run: npx norn ./tests/api.norn --sequence SmokeTest --json
```

## Syntax Reference

| Syntax | Description |
|--------|-------------|
| `var name = value` | Declare a variable |
| `{{name}}` | Reference a variable |
| `###` | Optional request separator |
| `sequence Name` | Start a sequence block |
| `end sequence` | End a sequence block |
| `var x = $1.path` | Capture value from response 1 |
| `run bash ./script.sh` | Run a bash script |
| `run powershell ./script.ps1` | Run a PowerShell script |
| `run js ./script.js` | Run a Node.js script |
| `var x = run js ./script.js` | Run script and capture output |
| `print Message` | Print a message |
| `print Title \| Body` | Print with expandable body |

## Keyboard Shortcuts

| Shortcut | Command |
|----------|---------|
| `Ctrl+Alt+R` / `Cmd+Alt+R` | Send Request |

## Extension Commands

- `Norn: Send Request` - Send the HTTP request at cursor
- `Norn: Run Sequence` - Run the sequence at cursor
- `Norn: Clear Cookies` - Clear all stored cookies
- `Norn: Show Stored Cookies` - Display cookies in output

## Requirements

- VS Code 1.108.1 or higher
- Node.js 22+ (for CLI and script execution)

## Release Notes

### 1.0.0

- HTTP request support (GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS)
- Variables with `var` and `{{}}`
- Sequences with response capture (`$N.path`)
- Script execution (`run bash/powershell/js`)
- Print statements with title and body
- Cookie jar with automatic persistence
- Full syntax highlighting
- IntelliSense for methods, headers, variables, keywords
- Diagnostic errors for undefined variables
- CLI for CI/CD pipelines
- JSON output mode for automation

## License

**Free for Personal Use** - You may use Norn for personal projects, learning, education, and non-commercial open-source projects at no cost.

**14-Day Commercial Evaluation** - Businesses may evaluate Norn free for 14 days before purchasing a license.

**Commercial Use Requires a License** - After the evaluation period, use within a business, by employees during work, or in CI/CD pipelines for commercial projects requires a license. Contact us for commercial licensing options.

See the [LICENSE](LICENSE) file for full terms.

---

**Enjoy using Norn!**
