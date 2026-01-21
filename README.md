# Job Worker Service

## Overview

A job worker service that provides an API to run arbitrary Linux processes on the host machine consisting of:

1. **Library**: A reusable Go package for managing job lifecycle and output streaming
2. **Server**: A gRPC API server with mTLS authentication
3. **CLI**: A command-line client for interacting with the server

## Target Level

This design targets Level 4 of the Teleport challenge.

Specifically, it focuses on:
- Secure gRPC API with mTLS
- Efficient multi-client output streaming
- Correct process lifecycle-handling

Resource isolation via cgroups and other Level 5 features are intentionally not included.

## Scope

### In-Scope
- Start/stop/status for processes
- Stream stdout/stderr from the beginning, even if you connect late
- Multiple clients streaming the same job simultaneously  
- mTLS auth, simple hardcoded authorization
- Everything in memory (jobs don't survive server restart)

### Out of Scope
- cgroups/resource limits
- stdin/interactive commands
- Persistence across restart
- High avaliability
- Metric endpoints
- Polling

## CLI User Experience


### Start a job
```bash
> jobctl start -- python3 script.py
Job started: abc123
```

### Check job status
```bash
> jobctl status abc123
ID:   abc123
Command:   python3 script.py
Status:   running
PID:   12345
Started:   2025-01-20T10:30:00Z
```

### Stream Output (like `tail -f`, blocks until job completes)
```bash
> jobctl logs abc123
[output streams here... until job ends]
```

### Stop a job
```bash
> jobctl stop abc123
Job stopped: abc123
```

### List all jobs
```bash
> jobctl list
ID   STATUS   COMMAND
abc123   stopped   python3 script.py
def456   running   /bin/sleep 1000
```

Connection config via flags:
```bash
> jobctl --server localhost:8443 --cert client.crt --key client.key --ca ca.crt start -- echo meow
```

## API Design

### gRPC Methods

- `Start(command[]) => job_id`: Start a new job with the given command and arguments
- `Stop(job_id)`: Stop a running job
- `Status(job_id) => JobStatus`: Get current status, PID, exit code, timestamps
- `StreamOutput(job_id) => stream bytes`: Stream output from job start until completion
- `List() => []JobStatus`: List all jobs

Output is streamed as raw bytes to support binary data. Possible statuses are: [`RUNNING`, `COMPLETED`, `FAILED`, `STOPPED`]

## TLS Configuration

### Certificate Setup

For the PoC, certs will be pre-generated and kept in static local files.

**Tradeoff**: Pre-generated certs simplify setup but should never be used in production. A production system would integrate with a proper PKI.

## Authorization Scheme

Auth is based on the client certificate's Common Name (CN). A hardcoded-mapping defines permissions:

```go
// TODO: In production, load from config file or external system
var permissions = map[string][]string{
    "admin":  {"start", "stop", "status", "logs", "list"},
    "viewer": {"status", "logs", "list"},
}
```

The server extracts the CN from the verified client certificate and checks against the permissions map. If the CN is not found or lacks the required permission, the request is denied with `PERMISSION_DENIED`.

**Tradeoff**: A production system would use a proper RBAC system with persistent storage or integrate with an external identity provider.

## Output Streaming

### Requirements
- Stream from start of process execution (not just new output)
- Support multiple concurrent clients
- No busy-waiting or polling
- Handle binary data (no text assumptions)

### Design

Each job maintains an in-memory buffer that captures combined stdout/stderr from process start. The buffer supports multiple concurrent readers, each tracking their own read offset so they can independently stream from the beginning.

To avoid polling, readers block using a condition variable (`sync.Cond`) when they've caught up to the latest output. When new data arrives, all waiting readers are awoken via broadcast. Then we can use the "`tail -f`"-like functionality without busy-waiting.

**Tradeoff**: Storing all output in-memory limits the size of output we can handle. A production system would use a file-backed buffer. For this PoC, this in-memory approach is simpler and sufficient for reasonable output sizes.

## TLS Configuration

This project will use mTLS with TLS 1.3 only. Go's TLS 1.3 implementation handles cipher suite negotiation automatically, limiting it to strong options (e.g. AES-128-GCM, AES-256-GCM, or ChaCha20-Poly1305). Server will be configured to require and verify client certificates.

For the PoC, certs will be pre-generated and kept in static local files.

**Tradeoff**: Pre-generated certs simplify setup but should never be used in production. A production system would integrate with a proper PKI.

## Process Lifecycle

### Starting a Job

1. Generate unique job ID (UUID)
2. Create output buffer
3. Fork process with `exec.Command`
4. Redirect stdout/stderr to output buffer
5. Store job metadata in in-memory store
6. Return job ID

### Stopping a Job

1. Send `SIGTERM` to process
2. Wait briefly for graceful shutdown
3. Send `SIGKILL` if still running
4. Mark job as stopped

### Job States

- `RUNNING`
- `COMPLETED` (exit code 0)
- `FAILED` (exit code != 0)
- `STOPPED` (killed by user)

## Testing Strategy

### Unit Tests
- Job state transitions
- Output buffer read/write with concurrent access
- Authorization permission checks

### Integration Tests
- Full job lifecycle (start => status => logs => stop)
- mTLS handshake success and failure (wrong cert, expired cert)
- Authorization denial for unpermitted operations
- Output streaming with multiple concurrent clients
- Graceful shutdown

Tests requiring network access will use localhost. Tests will be run with `-race` flag to detect data races.

## Tradeoffs

| Decision | Rationale | Production Alternative |
|----------|-----------|----------------------|
| In-memory job state | PoC | Persistent storage (SQLite, etc.) |
| In-memory output buffer | PoC | File-backed buffer for large outputs |
| Hardcoded authorization | Faster to implement | External RBAC system |
| Pre-generated certs | Easy setup for reviewers | PKI integration |
| Single server instance | Scope reduction | Distributed state, load balancing |

### In-memory Job State and Output Buffer
Everything is being kept in-memory like jobs, job status info, and output buffers. A real system would persist job state and stream output to disk so you don't lose everything on restart and don't run out of memory on large outputs.

### Authentication
Hardcoded authentication is not production-ready but demonstrates the access pattern without building out config loading with an external role-based access control system.

### Pre-generated Certs
Pre-generated certs are convenient for reviewers but a real deployment would have proper cert management with an encryption policy that issues digital certs like an internal CA (Vault) or Let's Encypt which would allow for rotation and secure storage and retrieval.