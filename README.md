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

Connection config via flags:
```bash
> jobctl --server localhost:8443 --cert client.crt --key client.key --ca ca.crt start -- echo meow
```

## API Design

### Protocol Buffer Specification

```protobuf
syntax = "proto3";

package jobworker.v1;

option go_package = "github.com/jobworker/api/v1;jobworkerv1";

service JobWorker {
  // Start a new job with the given command and arguments
  rpc Start(StartRequest) returns (StartResponse);
  
  // Stop a running job
  rpc Stop(StopRequest) returns (StopResponse);
  
  // Get current status of a job
  rpc Status(StatusRequest) returns (StatusResponse);
  
  // Stream output from job start until completion
  rpc StreamOutput(StreamOutputRequest) returns (stream StreamOutputResponse);
}

message StartRequest {
  // Command and arguments to execute (e.g., ["python3", "script.py"])
  repeated string command = 1;
}

message StartResponse {
  string job_id = 1;
}

message StopRequest {
  string job_id = 1;
}

message StopResponse {}

message StatusRequest {
  string job_id = 1;
}

message StatusResponse {
  string job_id = 1;
  repeated string command = 2;
  JobState state = 3;
  int32 pid = 4;
  int32 exit_code = 5;
  string started_at = 6;   // RFC3339 timestamp
  string completed_at = 7; // RFC3339 timestamp, empty if still running
}

enum JobState {
  JOB_STATE_UNSPECIFIED = 0;
  JOB_STATE_RUNNING = 1;
  JOB_STATE_COMPLETED = 2; // exit code 0
  JOB_STATE_FAILED = 3;    // exit code != 0
  JOB_STATE_STOPPED = 4;   // killed by user
}

message StreamOutputRequest {
  string job_id = 1;
}

message StreamOutputResponse {
  // Raw bytes of combined stdout/stderr
  bytes data = 1;
}
```

Output is streamed as raw bytes to support binary data.

## TLS Configuration

### Certificate Setup

This project uses mTLS with TLS 1.3 only. Go's TLS 1.3 implementation handles cipher suite negotiation automatically, limiting it to strong options (AES-128-GCM, AES-256-GCM, or ChaCha20-Poly1305). The server will be configured to require and verify client certificates.

For the PoC, certificates will be pre-generated

#### Certificate Generation

```bash
# Generate CA private key (ECDSA P-256)
openssl ecparam -genkey -name prime256v1 -out ca.key

# Generate self-signed CA certificate (valid 365 days)
openssl req -new -x509 -days 365 -key ca.key -out ca.crt \
  -subj "/CN=JobWorker CA"

# Generate server private key
openssl ecparam -genkey -name prime256v1 -out server.key

# Generate server CSR
openssl req -new -key server.key -out server.csr \
  -subj "/CN=localhost"

# Sign server certificate with CA
openssl x509 -req -days 365 -in server.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out server.crt \
  -extfile <(echo "subjectAltName=DNS:localhost,IP:127.0.0.1")

# Generate client private key (repeat for each client identity)
openssl ecparam -genkey -name prime256v1 -out admin.key

# Generate client CSR with CN used for authorization
openssl req -new -key admin.key -out admin.csr \
  -subj "/CN=admin"

# Sign client certificate with CA
openssl x509 -req -days 365 -in admin.csr -CA ca.crt -CAkey ca.key \
  -CAcreateserial -out admin.crt
```

**Algorithm choices:**
- **ECDSA with P-256**: Modern, efficient, and widely supported. Provides equivalent security to RSA-3072 with smaller key sizes.
- **SHA-256**: Used by OpenSSL for signing (and is the default for ECDSA).

**Tradeoff**: Pre-generated certs simplify setup but should never be used in production. A production system would integrate with a proper PKI.

## Authorization Scheme

Auth is based on the client certificate's Common Name (CN). A hardcoded-mapping defines permissions:

```go
// TODO: In production, load from config file or external system
var permissions = map[string][]string{
    "admin":  {"start", "stop", "status", "logs"},
    "viewer": {"status", "logs"},
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

## Process Lifecycle

### Starting a Job

1. Generate unique job ID (UUID)
2. Create output buffer
3. Fork process with `exec.Command`
4. Redirect stdout/stderr to output buffer
5. Store job metadata in in-memory store
6. Return job ID

### Stopping a Job

1. Send `SIGKILL` to process
2. Mark job as stopped

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