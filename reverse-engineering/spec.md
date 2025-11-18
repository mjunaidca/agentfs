# AgentFS Specification (Reverse Engineered)

**Version:** 0.1.2 (Current Implementation)
**Date:** 2025-11-18
**Status:** Alpha - Development/Testing/Experimentation Only

---

## Problem Statement

AI agents need persistent, auditable storage that traditional filesystems don't provide. Current solutions for agent state management suffer from:

- **Lack of Auditability**: No built-in tracking of what agents do with files and data
- **Poor Reproducibility**: Cannot snapshot and restore exact agent execution states
- **Fragmented Storage**: Agents scatter data across multiple storage mechanisms (files, databases, caches)
- **No Tool Tracking**: Unable to analyze which tool calls succeed, fail, or cause performance issues
- **Deployment Complexity**: Multi-file dependencies make agents hard to version control and distribute

**Core Insight**: Just as traditional filesystems abstract storage for applications, AI agents need specialized storage abstractions that understand agent-specific requirements: auditability, reproducibility, and portability.

---

## System Intent

### Target Users
- **AI Agent Developers** - Building autonomous agents with Claude SDK, Mastra, OpenAI Agents, or custom frameworks
- **Research Scientists** - Experimenting with agent architectures requiring reproducible state
- **DevOps Engineers** - Deploying agents with version-controlled, portable storage
- **Security Auditors** - Analyzing agent behavior through complete audit trails

### Core Value Proposition

**"The entire agent runtime—files, state, history—in a single SQLite file."**

AgentFS provides:
1. **Single-file portability** - Move/version/deploy agents by copying one file
2. **Complete auditability** - Every file operation and tool call permanently recorded
3. **Perfect reproducibility** - Snapshot and restore exact execution states with `cp`
4. **Zero configuration** - No database servers, no distributed systems, just SQLite
5. **Framework agnostic** - Works with any AI framework through simple SDK

### Why This Exists Instead of Alternatives

| Alternative | Limitation | AgentFS Solution |
|-------------|-----------|------------------|
| Regular filesystem | No audit trail, scattered files | SQLite-based with built-in tracking |
| PostgreSQL/MySQL | Requires server, complex deployment | Single file, no server needed |
| Redis/Memcached | Volatile, no persistence guarantees | ACID transactions, durable storage |
| Cloud object storage | Network dependency, no local development | Works offline, local-first |
| Git for state | Not designed for runtime data | Optimized for agent runtime patterns |

---

## Functional Requirements

### FR-1: Virtual Filesystem Operations
**What**: POSIX-like filesystem for agent file artifacts
**Why**: Agents need familiar file operations (read, write, mkdir) for outputs, caches, and working data
**Inputs**: Path (string), content (bytes or text)
**Outputs**: File content, directory listings, metadata
**Success Criteria**:
- ✅ Create, read, update, delete files
- ✅ Create and list directories
- ✅ Support symbolic links
- ✅ Support hard links (multiple paths to same file)
- ✅ POSIX-like metadata (permissions, timestamps, ownership)
- ✅ Path resolution with absolute and relative paths
- ✅ Automatic parent directory creation

**Operations**:
```typescript
// File operations
await agent.fs.writeFile('/output/report.pdf', pdfBuffer)
const content = await agent.fs.readFile('/output/report.pdf')
await agent.fs.deleteFile('/output/report.pdf')

// Directory operations
await agent.fs.mkdir('/artifacts')
const files = await agent.fs.readdir('/artifacts')

// Metadata
const stats = await agent.fs.stat('/output/report.pdf')
console.log(stats.size, stats.mtime)

// Symbolic links
await agent.fs.symlink('/data/source.txt', '/link.txt')
const target = await agent.fs.readlink('/link.txt')
```

### FR-2: Key-Value State Management
**What**: Simple get/set operations for agent context and preferences
**Why**: Not all agent state fits the filesystem model (session data, preferences, counters)
**Inputs**: Key (string), value (any JSON-serializable data)
**Outputs**: Retrieved value or undefined
**Success Criteria**:
- ✅ Set values with automatic JSON serialization
- ✅ Get values with automatic JSON deserialization
- ✅ Delete keys
- ✅ List all keys
- ✅ Automatic timestamp tracking (created_at, updated_at)
- ✅ Upsert semantics (insert or update)

**Operations**:
```typescript
// Simple values
await agent.kv.set('counter', 42)
const count = await agent.kv.get('counter') // 42

// Complex objects
await agent.kv.set('user:preferences', { theme: 'dark', lang: 'en' })
const prefs = await agent.kv.get('user:preferences')

// Deletion
await agent.kv.delete('counter')

// List keys
const allKeys = await agent.kv.keys()
```

### FR-3: Tool Call Audit Trail
**What**: Record all tool invocations with timing, parameters, and results
**Why**: Debug agent failures, analyze performance, audit compliance
**Inputs**: Tool name, parameters, execution timing, result or error
**Outputs**: Tool call records, performance statistics
**Success Criteria**:
- ✅ Record tool calls with start/completion timestamps
- ✅ Track both successful and failed calls
- ✅ Store parameters and results as JSON
- ✅ Calculate duration automatically
- ✅ Query by tool name
- ✅ Query by time range
- ✅ Aggregate statistics (success rate, average duration)
- ✅ Insert-only audit log (immutable history)

**Operations**:
```typescript
// Record successful call
await agent.tools.record(
  'web_search',
  startTime,
  endTime,
  { query: 'AgentFS' },
  { results: [...] }
)

// Record failed call
await agent.tools.record(
  'database_query',
  startTime,
  endTime,
  { sql: 'SELECT ...' },
  undefined,
  'Connection timeout'
)

// Query and analyze
const searches = await agent.tools.getByName('web_search')
const stats = await agent.tools.getStats()
console.log(`Success rate: ${stats.successful / stats.total_calls}`)
```

### FR-4: CLI Management Interface
**What**: Command-line tool for creating and inspecting agent filesystems
**Why**: Developers need to initialize databases and debug agent state externally
**Inputs**: Commands and paths
**Outputs**: Created databases, file listings, file content
**Success Criteria**:
- ✅ `agentfs init [filename]` - Create new agent database
- ✅ `agentfs fs ls [path]` - List files in agent filesystem
- ✅ `agentfs fs cat <path>` - Display file contents
- ✅ Force overwrite with `--force` flag
- ✅ Works without running agent code

**Operations**:
```bash
# Initialize
$ agentfs init agent.db
Created agent filesystem: agent.db

# Inspect from outside
$ agentfs fs ls /
f hello.txt
d output

$ agentfs fs cat /hello.txt
hello from agent
```

### FR-5: Sandboxed Execution (Linux Only)
**What**: Run programs with agent filesystem mounted at `/agent`
**Why**: Agents need to execute arbitrary code safely with controlled filesystem access
**Inputs**: Command to execute, mount configurations
**Outputs**: Intercepted filesystem operations, process execution
**Success Criteria**:
- ✅ Mount agent.db at `/agent` path
- ✅ Intercept all filesystem syscalls
- ✅ Support bind mounts for host directories
- ✅ Optional strace-like debugging output
- ✅ Per-process file descriptor isolation
- ✅ Transparent operation (programs see normal filesystem)

**Operations**:
```bash
# Basic sandbox
$ agentfs run /bin/bash
$ echo "test" > /agent/file.txt
$ cat /agent/file.txt
test

# With custom mounts
$ agentfs run --mount type=bind,src=/data,dst=/data python script.py

# Debug mode
$ agentfs run --strace python script.py
[strace] openat(AT_FDCWD, "/agent/file.txt", O_RDONLY) = 3
```

### FR-6: Multi-Language SDK Support
**What**: SDKs for Rust and TypeScript/JavaScript
**Why**: Support both system-level (Rust) and web/Node.js (TypeScript) ecosystems
**Inputs**: Database path
**Outputs**: AgentFS instance with fs/kv/tools interfaces
**Success Criteria**:
- ✅ Feature parity between Rust and TypeScript
- ✅ Async/await API in both languages
- ✅ Comprehensive type definitions
- ✅ Error handling with descriptive messages
- ✅ NPM and crates.io distribution

---

## Non-Functional Requirements

### NFR-1: Auditability
**Requirement**: Complete, queryable history of all agent operations
**Evidence**:
- Tool calls table is insert-only (no updates/deletes)
- All timestamps use Unix epoch format
- Parameters and results stored as JSON for SQL queries
- File operations tracked via inode metadata
- Example audit query:
  ```sql
  SELECT name, COUNT(*) as calls, AVG(duration_ms) as avg_duration
  FROM tool_calls
  WHERE started_at > unixepoch() - 86400
  GROUP BY name
  ```

### NFR-2: Reproducibility
**Requirement**: Snapshot and restore exact agent states
**Evidence**:
- Single SQLite file contains all state
- ACID transactions ensure consistency
- Snapshotting is simple file copy: `cp agent.db snapshot.db`
- No external dependencies for state restoration
- Examples include snapshot workflows

### NFR-3: Portability
**Requirement**: Move agents between machines without reconfiguration
**Evidence**:
- Single file contains everything
- SQLite is cross-platform
- No hardcoded absolute paths in database
- Works with Turso for remote/cloud deployment
- Supported platforms: Linux (x86_64, ARM64), macOS (ARM64, x86_64), Windows (x86_64)

### NFR-4: Performance
**Requirement**: Efficient for agent workloads (many small files, frequent KV updates)
**Evidence**:
- Inode-based filesystem separates metadata from data
- Chunked file storage supports large files
- Indexes on critical fields (parent_ino, started_at, key)
- Connection pooling via Arc<Connection>
- Async I/O throughout with Tokio

### NFR-5: Security (Sandbox)
**Requirement**: Isolated execution environment for untrusted agent code
**Evidence**:
- Linux sandbox uses ptrace via Reverie framework
- Syscall interception at kernel boundary
- Mount table controls accessible paths
- Per-process file descriptor isolation
- Optional strace mode for debugging

### NFR-6: Developer Experience
**Requirement**: Simple, intuitive API that "just works"
**Evidence**:
- Zero configuration (no config files)
- In-memory mode for testing (`:memory:`)
- Familiar POSIX-like filesystem API
- Automatic parent directory creation
- Comprehensive error messages
- Working examples for 3 AI frameworks

### NFR-7: Observability
**Requirement**: Built-in visibility into agent behavior
**Evidence**:
- Tool call statistics API
- File metadata with timestamps
- KV store created_at/updated_at tracking
- Optional strace mode for syscall debugging
- Direct SQL query support for custom analytics

---

## Constraints

### Technical Constraints

**Database**:
- ✅ SQLite 3.x compatible (via Turso 0.3.2)
- ✅ Turso (libSQL fork) provides extended features
- ✅ Single-threaded writes (SQLite limitation)
- ✅ Maximum database size limited by filesystem (not SQLite)

**Platform Support**:
- ✅ **Full support**: Linux x86_64, macOS ARM64, macOS x86_64
- ⚠️ **SDK only**: Windows (no sandbox)
- ⚠️ **Sandbox requires Linux**: Reverie ptrace framework

**Language/Runtime**:
- ✅ Rust 2021 edition or later
- ✅ Node.js 16+ for TypeScript SDK
- ✅ Tokio async runtime (Rust)

### Design Constraints

**Schema Immutability**:
- Schema follows SPEC.md v0.0 specification
- Breaking changes require major version bump
- Extension points via separate tables

**Filesystem Design**:
- Root inode MUST be ino=1
- Inode-based design (not path-based)
- POSIX semantics for compatibility

**Audit Trail**:
- Tool calls are append-only
- No deletion of historical records
- Must accept eventual large database size

### Integration Constraints

**AI Frameworks Tested**:
- ✅ Anthropic Claude Agent SDK
- ✅ Mastra AI framework
- ✅ OpenAI Agents SDK
- Framework-agnostic design (no framework lock-in)

**Deployment Contexts**:
- ✅ Local development (file-based)
- ✅ Cloud deployment (Turso remote databases)
- ✅ CI/CD pipelines (in-memory for tests)
- ⚠️ Not designed for multi-agent concurrent writes (single SQLite file)

---

## Non-Goals & Out of Scope

**Explicitly excluded** (based on missing implementation):

❌ **Distributed agent coordination**
- No multi-agent locking or coordination
- Single SQLite file = single agent primary use case
- Alternative: Use separate databases per agent

❌ **Real-time streaming of large files**
- Chunked storage supports large files, but not optimized for streaming
- Files loaded entirely into memory

❌ **Advanced permission model**
- Basic POSIX permissions stored, but not enforced
- Sandbox provides process isolation, not per-file ACLs

❌ **Built-in encryption at rest**
- SQLite file stored in plaintext
- Alternative: Use SQLite encryption extension or encrypted filesystem

❌ **Automatic cloud sync**
- No built-in synchronization across Turso databases
- Manual: Use Turso replication features

❌ **Windows sandbox support**
- Reverie ptrace requires Linux
- Windows users can use SDK without sandbox

❌ **Garbage collection / automatic archival**
- Tool call history grows unbounded
- User responsibility to archive old data

---

## Known Gaps & Technical Debt

### Gap 1: Limited Error Recovery in Filesystem Operations
**Issue**: Partial writes or transaction failures may leave filesystem in inconsistent state
**Evidence**: `/home/user/agentfs/sdk/rust/src/filesystem.rs:200-250` - No explicit rollback on multi-step operations
**Impact**: Edge cases (disk full, process kill) could corrupt database
**Recommendation**: Wrap multi-step operations in explicit transactions with rollback

### Gap 2: No Built-in Quota Management
**Issue**: Agent can fill unlimited disk space
**Evidence**: No size limits in schema or SDK
**Impact**: Runaway agents could exhaust disk
**Recommendation**: Add optional quota tracking (extension point in spec)

### Gap 3: Missing Symbolic Link Resolution in Some Operations
**Issue**: Not all filesystem operations follow symlinks
**Evidence**: `lstat` returns symlink metadata, but not all ops check for symlinks first
**Impact**: Inconsistent behavior between symlink and target
**Recommendation**: Standardize symlink resolution policy

### Gap 4: Sparse Testing on Non-Linux Platforms
**Issue**: Primary testing on Linux, limited macOS/Windows validation
**Evidence**: CI/CD workflows focus on Linux builds
**Impact**: Platform-specific bugs may exist
**Recommendation**: Expand test matrix to all supported platforms

### Gap 5: No Migration Path for Schema Changes
**Issue**: Breaking schema changes require manual database migration
**Evidence**: No migration framework in codebase
**Impact**: Version upgrades may break existing databases
**Recommendation**: Add Alembic-style migration system

### Gap 6: Limited Documentation for Remote Turso Setup
**Issue**: Examples focus on local SQLite, not cloud Turso deployment
**Evidence**: `/home/user/agentfs/MANUAL.md:548-568` mentions Turso but lacks complete guide
**Impact**: Friction deploying agents to production
**Recommendation**: Add production deployment guide with Turso configuration

### Gap 7: No Built-in Compression for Large Files
**Issue**: Large files (PDFs, images) stored as-is, wasting space
**Evidence**: `fs_data.data` stores raw BLOBs
**Impact**: Database size larger than necessary
**Recommendation**: Add optional compression (extension point in spec)

---

## Tech Debt Prioritization

| Priority | Debt Item | Impact | Effort | ROI |
|----------|-----------|--------|--------|-----|
| **P0** | Transaction safety in filesystem ops | High (data corruption risk) | Medium | High |
| **P1** | Schema migration framework | Medium (upgrade friction) | High | Medium |
| **P2** | Quota management system | Medium (resource exhaustion) | Medium | Medium |
| **P3** | Production Turso deployment guide | Low (workaround exists) | Low | High |
| **P4** | Comprehensive platform testing | Medium (platform bugs) | High | Medium |
| **P5** | Built-in compression support | Low (optional optimization) | High | Low |

---

## Success Criteria

The implementation SUCCEEDS when:

✅ **Completeness**: All functional requirements (FR-1 through FR-6) are implemented
✅ **Portability**: Single file (`cp agent.db`) moves entire agent state
✅ **Auditability**: SQL queries reveal complete agent history
✅ **Reproducibility**: Snapshot restores exact execution state
✅ **Simplicity**: Zero configuration for basic use cases
✅ **Framework Integration**: Works with 3+ AI frameworks without modifications
✅ **Developer Experience**: Examples run successfully out-of-the-box

The implementation NEEDS IMPROVEMENT when:

⚠️ Transactions fail without rollback causing corruption
⚠️ Agents exhaust disk space without warnings
⚠️ Platform-specific bugs discovered in production
⚠️ Schema upgrades break existing databases

---

## Regeneration Test

**Question**: Can another developer read this spec and build equivalent system?

**Answer**: ✅ **YES** - This spec provides:
- Complete problem statement and motivation
- All functional requirements with input/output examples
- Non-functional requirements with evidence from codebase
- Constraints and platform limitations
- Known gaps and technical debt
- Success criteria for validation

**Missing for 100% regeneration**:
- Detailed SQLite schema (→ See `/home/user/agentfs/SPEC.md` for complete schema)
- Specific Reverie ptrace integration details (→ See plan.md for architecture)
- Turso connection configuration (→ See examples in `/home/user/agentfs/examples/`)

---

## Version History

- **0.1.2** (Current) - Production alpha with examples for 3 AI frameworks
- **0.1.1** - Initial public release
- **0.1.0** - Internal development version

---

**Next Steps for Regeneration**:
1. Read `plan.md` for architecture and design patterns
2. Read `tasks.md` for step-by-step implementation guide
3. Read `intelligence-object.md` for reusable architectural decisions
4. Reference `SPEC.md` for complete SQLite schema details
