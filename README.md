# pi-sandbox

OS-level sandboxing for pi's bash tool, plus path policy enforcement for the
read, write, and edit tools. When a blocked action is attempted, the user is
prompted to allow it temporarily or permanently rather than silently failing.

## What it does

**Bash commands** are wrapped with `sandbox-exec` (macOS) or `bubblewrap`
(Linux) to enforce network and filesystem restrictions at the OS level.

**Read, write, and edit tool calls** are intercepted before execution and
checked against the same filesystem policy. The OS-level sandbox cannot cover
these tools because they run directly in the Node.js process rather than in a
subprocess.

When a block is triggered, a prompt appears with four options:

- Abort (keep blocked)
- Allow for this session only
- Allow for this project — written to `.pi/sandbox.json`
- Allow for all projects — written to `~/.pi/agent/sandbox.json`

**Session allowances** are held in memory only. They are never written to disk
and the agent has no way to read or modify them. They are reset when the
extension reloads or pi restarts.

### What is prompted vs. hard-blocked

| Rule | Behaviour |
|------|-----------|
| Domain not in `allowedDomains` | Prompted (bash and `!cmd`) |
| Path not in `allowWrite` | Prompted (write/edit tools and bash write failures) |
| Path in `denyRead` | Hard-blocked, no prompt |
| Path in `denyWrite` | Hard-blocked, no prompt |
| Domain in `deniedDomains` | Hard-blocked at OS level, no prompt |

If a path is added to `allowWrite` via a prompt but is also present in
`denyWrite`, it remains blocked. A warning is shown explaining which config
files to check.

## Installation

```bash
pi install git:github.com/your-org/pi-sandbox
```

Or from a local path:

```bash
pi install ./path/to/sandbox
```

After installing, run `npm install` inside the extension directory to fetch the
`@anthropic-ai/sandbox-runtime` dependency. On Linux, also install `bubblewrap`,
`socat`, and `ripgrep`.

## Configuration

Two config files are merged, with the project file taking precedence:

- `~/.pi/agent/sandbox.json` — global, applies to all projects
- `.pi/sandbox.json` — project-local, overrides global

```json
{
  "enabled": true,
  "network": {
    "allowedDomains": ["github.com", "*.github.com"],
    "deniedDomains": []
  },
  "filesystem": {
    "denyRead": ["~/.ssh", "~/.aws", "~/.gnupg"],
    "allowWrite": [".", "/tmp"],
    "denyWrite": [".env", ".env.*", "*.pem", "*.key"]
  }
}
```

`allowedDomains` supports `*.example.com` wildcards. `allowWrite` uses prefix
matching, so `.` covers the entire current working directory. `denyWrite` takes
precedence over `allowWrite`.

If neither file exists, built-in defaults apply (see above for the defaults).

## Usage

```
pi --no-sandbox          disable sandboxing for the session
/sandbox                 show current configuration and session allowances
```

The footer shows a lock indicator while the sandbox is active.

## Ackowledgements
Based on code from
[badlogic/pi-mono](https://github.com/badlogic/pi-mono/blob/main/packages/coding-agent/examples/extensions/sandbox/index.ts)
by Mario Zechner, used under the
[MIT License](https://github.com/badlogic/pi-mono/blob/main/LICENSE).
