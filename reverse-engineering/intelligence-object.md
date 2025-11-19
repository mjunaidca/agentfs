# AgentFS Reusable Intelligence (Reverse Engineered)

**Version:** 0.1.2
**Date:** 2025-11-18
**Purpose:** Extract reusable patterns, skills, and architectural decisions from AgentFS

---

## Extracted Reusable Skills

### Skill 1: SQLite-as-Filesystem Pattern

**Persona**: You are a systems architect designing persistent storage for applications that need filesystem semantics but want single-file portability.

**When to apply**:
- Application needs file/directory abstractions
- Deployment requires single-file portability
- Need ACID transactions for filesystem operations
- Want SQL query capability over file metadata
- Target is single-writer or controlled concurrent access

**Questions to ask before implementing**:

1. **What filesystem semantics are required?**
   - Full POSIX (hard links, symlinks, permissions)?
   - Simplified (just files and directories)?
   - Read-only or read-write?

2. **What access patterns dominate?**
   - Many small files or few large files?
   - Sequential reads or random access?
   - Metadata queries or content retrieval?

3. **What level of POSIX compatibility is needed?**
   - Inode numbers and link counts?
   - File permissions enforcement?
   - Extended attributes?

4. **What are the performance requirements?**
   - Acceptable latency for file operations?
   - Expected database size?
   - Concurrent access patterns?

**Principles**:

- **Separate namespace from data**: Use inode design (fs_inode, fs_dentry, fs_data)
  - Enables hard links (multiple names → same inode)
  - Makes renames efficient (update dentry, not data)
  - Allows metadata queries without reading file content

- **Chunk large files**: Store file content in chunks with offset
  - Supports files larger than SQLite BLOB limit
  - Enables streaming for large files
  - Makes partial updates possible (though not implemented here)

- **Use indexes strategically**: Index on query hot paths
  - `idx_fs_dentry_parent` for directory listings
  - `idx_fs_data_ino_offset` for file reads
  - Balance index overhead vs query speed

- **Ensure root directory always exists**: inode 1 as root
  - Simplifies path resolution (always start at 1)
  - Prevents "no root directory" errors
  - Initialize on database creation

**Implementation Pattern**:

```rust
// From /home/user/agentfs/sdk/rust/src/filesystem.rs

// 1. Schema Design
const CREATE_FS_INODE: &str = "
CREATE TABLE IF NOT EXISTS fs_inode (
  ino INTEGER PRIMARY KEY AUTOINCREMENT,
  mode INTEGER NOT NULL,      -- File type + permissions (0o100644 = regular file)
  uid INTEGER, gid INTEGER,
  size INTEGER NOT NULL,
  atime INTEGER, mtime INTEGER, ctime INTEGER  -- Unix timestamps
)";

const CREATE_FS_DENTRY: &str = "
CREATE TABLE IF NOT EXISTS fs_dentry (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,
  parent_ino INTEGER NOT NULL,
  ino INTEGER NOT NULL,
  UNIQUE(parent_ino, name)    -- No duplicate names in directory
)";

const CREATE_FS_DATA: &str = "
CREATE TABLE IF NOT EXISTS fs_data (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  ino INTEGER NOT NULL,
  offset INTEGER NOT NULL,    -- Byte offset in file
  size INTEGER NOT NULL,
  data BLOB NOT NULL
)";

// 2. Path Resolution
async fn resolve_path(&self, path: &str) -> Result<Option<i64>> {
    if path == "/" {
        return Ok(Some(ROOT_INO));  // ino=1
    }

    let mut current_ino = ROOT_INO;
    for component in path.split('/').filter(|s| !s.is_empty()) {
        // Traverse directory tree
        let result = self.conn.query(
            "SELECT ino FROM fs_dentry WHERE parent_ino = ? AND name = ?",
            (current_ino, component)
        ).await?;

        if result.rows.is_empty() {
            return Ok(None);  // Path not found
        }

        current_ino = result.rows[0].get::<i64>(0)?;
    }
    Ok(Some(current_ino))
}

// 3. Write File
async fn write_file(&self, path: &str, data: &[u8]) -> Result<()> {
    // a. Ensure parent directories exist
    self.mkdir_parents(path).await?;

    // b. Resolve or create inode
    let ino = match self.resolve_path(path).await? {
        Some(ino) => {
            // File exists, delete old data
            self.conn.execute("DELETE FROM fs_data WHERE ino = ?", (ino,)).await?;
            ino
        }
        None => {
            // Create new inode
            let result = self.conn.query(
                "INSERT INTO fs_inode (mode, size, atime, mtime, ctime)
                 VALUES (?, 0, unixepoch(), unixepoch(), unixepoch())
                 RETURNING ino",
                (DEFAULT_FILE_MODE,)  // 0o100644
            ).await?;

            let ino = result.rows[0].get::<i64>(0)?;

            // Create dentry
            let (parent_path, name) = split_path(path);
            let parent_ino = self.resolve_path(parent_path).await?.unwrap();

            self.conn.execute(
                "INSERT INTO fs_dentry (name, parent_ino, ino) VALUES (?, ?, ?)",
                (name, parent_ino, ino)
            ).await?;

            ino
        }
    };

    // c. Insert data chunk
    self.conn.execute(
        "INSERT INTO fs_data (ino, offset, size, data) VALUES (?, 0, ?, ?)",
        (ino, data.len(), data)
    ).await?;

    // d. Update inode size
    self.conn.execute(
        "UPDATE fs_inode SET size = ?, mtime = unixepoch() WHERE ino = ?",
        (data.len(), ino)
    ).await?;

    Ok(())
}

// 4. Read File
async fn read_file(&self, path: &str) -> Result<Option<Vec<u8>>> {
    let ino = match self.resolve_path(path).await? {
        Some(ino) => ino,
        None => return Ok(None),
    };

    // Fetch all chunks ordered by offset
    let rows = self.conn.query(
        "SELECT data FROM fs_data WHERE ino = ? ORDER BY offset ASC",
        (ino,)
    ).await?;

    // Concatenate chunks
    let mut content = Vec::new();
    for row in rows.rows {
        let chunk = row.get::<Vec<u8>>(0)?;
        content.extend_from_slice(&chunk);
    }

    // Update access time
    self.conn.execute(
        "UPDATE fs_inode SET atime = unixepoch() WHERE ino = ?",
        (ino,)
    ).await?;

    Ok(Some(content))
}
```

**When to apply this pattern**:
- ✅ Single-user applications needing filesystem-like storage
- ✅ Embedded systems with limited storage
- ✅ Development tools requiring portable project files
- ✅ AI agents with file artifacts
- ❌ Multi-writer high-concurrency scenarios
- ❌ Streaming large video files (SQLite not optimized for this)
- ❌ Applications requiring mmap() for performance

**Lessons Learned**:
- Inode design adds complexity but enables POSIX semantics
- Chunking is necessary for large files (SQLite BLOB limit)
- Path resolution is O(n) where n = path depth; cache for performance
- Root directory initialization prevents edge cases

---

### Skill 2: Dual-Language SDK Strategy

**Persona**: You are a platform engineer building a library that must support multiple programming ecosystems without fragmenting the user base.

**When to apply**:
- Library serves developers in multiple language ecosystems
- Core logic is database or file-based (language-agnostic storage)
- Need to balance maintenance overhead vs market reach
- Different languages serve different use cases (systems vs web)

**Questions to ask before implementing**:

1. **What languages cover 80%+ of target users?**
   - Identify primary language ecosystems
   - Consider framework lock-in (TypeScript for web, Rust for systems)
   - Evaluate community size and momentum

2. **Can languages share the same underlying storage?**
   - File formats compatible across languages?
   - Database drivers available for both?
   - Binary protocols well-defined?

3. **What is the maintenance model?**
   - Can one language be canonical and others bindings?
   - Or must implementations be parallel?
   - How to ensure feature parity?

4. **What are the trade-offs?**
   - Code duplication vs FFI complexity
   - Type safety vs interoperability
   - Development speed vs consistency

**Principles**:

- **Identical API surface**: Same function names, parameters, return types
  - Users switching languages face zero learning curve
  - Documentation applies equally to both
  - Examples are translatable 1:1

- **Shared storage layer**: Both SDKs read/write same database schema
  - No "TypeScript databases" vs "Rust databases"
  - Cross-language tools (write in Rust, read in TypeScript)
  - Enables polyglot applications

- **Language-native idioms**: Don't force Rust patterns on TypeScript
  - Rust: `Result<T, E>`, ownership, explicit lifetimes
  - TypeScript: `Promise<T>`, exceptions, garbage collection
  - Same logic, different expression

- **Comprehensive testing**: Test cross-language compatibility
  - Write with Rust, read with TypeScript
  - Verify schema produced by both SDKs is identical
  - Performance parity (both should be fast)

**Implementation Pattern**:

```rust
// Rust SDK (/home/user/agentfs/sdk/rust/src/lib.rs)

pub struct AgentFS {
    conn: Arc<Connection>,
    pub kv: KvStore,
    pub fs: Filesystem,
    pub tools: ToolCalls,
}

impl AgentFS {
    pub async fn new(db_path: &str) -> Result<Self> {
        let db = Builder::new_local(db_path).build().await?;
        let conn = Arc::new(db.connect()?);

        let kv = KvStore::from_connection(conn.clone()).await?;
        let fs = Filesystem::from_connection(conn.clone()).await?;
        let tools = ToolCalls::from_connection(conn.clone()).await?;

        Ok(Self { conn, kv, fs, tools })
    }
}

// Usage
let agentfs = AgentFS::new("agent.db").await?;
agentfs.kv.set("key", &value).await?;
let data = agentfs.kv.get::<String>("key").await?;
```

```typescript
// TypeScript SDK (/home/user/agentfs/sdk/typescript/src/index.ts)

export class AgentFS {
    public readonly kv: KvStore;
    public readonly fs: Filesystem;
    public readonly tools: ToolCalls;

    private constructor(private db: Database) {
        this.kv = new KvStore(db);
        this.fs = new Filesystem(db);
        this.tools = new ToolCalls(db);
    }

    static async open(dbPath: string = ':memory:'): Promise<AgentFS> {
        const db = new Database(dbPath);
        const agentfs = new AgentFS(db);

        await agentfs.kv.initialize();
        await agentfs.fs.initialize();
        await agentfs.tools.initialize();

        return agentfs;
    }
}

// Usage
const agentfs = await AgentFS.open('agent.db');
await agentfs.kv.set('key', value);
const data = await agentfs.kv.get('key');
```

**Mapping Patterns**:

| Rust Pattern | TypeScript Equivalent | Reasoning |
|--------------|----------------------|-----------|
| `Result<T, E>` | `Promise<T>` + exceptions | TypeScript uses exceptions for errors |
| `async fn` | `async function` | Both have async/await |
| `Option<T>` | `T \| undefined` | TypeScript nullable types |
| `Arc<Connection>` | Garbage collected `Database` | No manual memory management in TS |
| Generic `<T>` | Generic `<T>` | Both support generics |
| `serde_json::to_string()` | `JSON.stringify()` | Built-in JSON in both |

**When to apply this pattern**:
- ✅ Library with clear abstraction layer (database, files, network)
- ✅ Languages have good database drivers (e.g., SQLite)
- ✅ Team has expertise in both languages
- ✅ Market demands both ecosystems (web + systems)
- ❌ One language dominates 95%+ of use cases
- ❌ FFI is available and simpler (C bindings to one language)
- ❌ Rapid iteration needed (maintaining two codebases is slow)

**Lessons Learned**:
- Start with one language, prove the design, then port
- Automated tests prevent divergence (run same test suite)
- Documentation should be language-neutral where possible
- Version numbers must stay in sync

---

### Skill 3: Syscall Interception for Virtual Filesystems (Linux)

**Persona**: You are a security engineer or sandbox architect needing to intercept and control filesystem access for untrusted code.

**When to apply**:
- Need to run untrusted or user-provided code
- Want to redirect filesystem operations to custom storage
- Require complete visibility into all file I/O
- Linux platform acceptable (ptrace is Linux-only)

**Questions to ask before implementing**:

1. **What level of isolation is required?**
   - Full process isolation (containers)?
   - Filesystem-only isolation (VFS)?
   - Network isolation needed too?

2. **What performance overhead is acceptable?**
   - Ptrace adds ~10-50% overhead
   - FUSE adds ~20-30% overhead
   - LD_PRELOAD adds ~1-5% overhead (but bypassable)

3. **What syscalls need interception?**
   - Just file operations (open, read, write)?
   - Network too (socket, connect)?
   - Process creation (fork, exec)?

4. **What compatibility is needed?**
   - Static binaries (ptrace works, LD_PRELOAD doesn't)?
   - setuid binaries (ptrace restricted)?
   - Multi-threaded applications?

**Principles**:

- **Intercept at kernel boundary**: Ptrace intercepts syscalls before kernel executes
  - Cannot be bypassed by user code
  - Complete visibility (all syscalls visible)
  - Works with static binaries and any language

- **Virtualize file descriptors**: Maintain separate FD table per process
  - Virtual FDs don't conflict with real FDs
  - Allows mixing virtual and real file access
  - Per-process isolation

- **Longest-prefix mount matching**: Mount table determines VFS backend
  - `/agent/*` → SqliteVfs
  - `/data/*` → BindVfs (passthrough to host)
  - `/etc/*` → Real filesystem (no interception)
  - Mimics Unix mount behavior

- **Handle all FD lifecycle events**: open, close, dup, fork
  - FD leaks lead to bugs
  - Fork duplicates FDs (copy FD table on fork)
  - Exec closes non-CLOEXEC FDs

**Implementation Pattern**:

```rust
// From /home/user/agentfs/sandbox/src/sandbox/mod.rs

use reverie::{Tool, Guest, Syscall};
use std::sync::{OnceLock, Mutex};

// Global state (one mount table for all processes)
static MOUNT_TABLE: OnceLock<MountTable> = OnceLock::new();
static FD_TABLES: OnceLock<Mutex<HashMap<Pid, FdTable>>> = OnceLock::new();

#[derive(Debug, Default, Clone)]
pub struct Sandbox;

#[reverie::tool]
impl Tool for Sandbox {
    async fn handle_syscall_event<T: Guest<Self>>(
        &self,
        guest: &mut T,
        syscall: Syscall,
    ) -> Result<i64, reverie::Error> {
        let mount_table = get_mount_table();
        let mut fd_table = get_fd_table(guest.tid().as_raw());

        // Dispatch based on syscall number
        let result = match syscall.number() as i64 {
            SYS_openat => handle_openat(&syscall, mount_table, &mut fd_table).await,
            SYS_read => handle_read(&syscall, &mut fd_table).await,
            SYS_write => handle_write(&syscall, &mut fd_table).await,
            SYS_close => handle_close(&syscall, &mut fd_table).await,
            SYS_stat | SYS_lstat => handle_stat(&syscall, mount_table).await,
            SYS_getdents64 => handle_getdents64(&syscall, mount_table, &mut fd_table).await,
            // ... more syscalls
            _ => {
                // Pass through to real kernel
                return Ok(guest.inject(syscall).await?);
            }
        };

        match result {
            Ok(ret) => Ok(ret),
            Err(e) => {
                // Map error to errno
                Ok(-libc::ENOENT as i64)  // Or appropriate errno
            }
        }
    }
}

// VFS abstraction
#[async_trait]
pub trait Vfs: Send + Sync {
    async fn open(&self, path: &str, flags: i32, mode: u32) -> VfsResult<VirtualFile>;
    async fn stat(&self, path: &str) -> VfsResult<VfsStat>;
    async fn readdir(&self, path: &str) -> VfsResult<Vec<DirEntry>>;
}

// Mount table (longest prefix match)
pub struct MountTable {
    mounts: Vec<(PathBuf, Arc<dyn Vfs>)>,  // Sorted by path length, descending
}

impl MountTable {
    pub fn find(&self, path: &Path) -> Option<(&Path, &Arc<dyn Vfs>, &Path)> {
        for (mount_point, vfs) in &self.mounts {
            if path.starts_with(mount_point) {
                let rel_path = path.strip_prefix(mount_point).unwrap();
                return Some((mount_point, vfs, rel_path));
            }
        }
        None
    }
}

// File descriptor table (per process)
pub struct FdTable {
    fds: HashMap<i32, VirtualFile>,
    next_fd: i32,
}

impl FdTable {
    pub fn insert(&mut self, file: VirtualFile) -> i32 {
        let fd = self.next_fd;
        self.fds.insert(fd, file);
        self.next_fd += 1;
        fd
    }

    pub fn remove(&mut self, fd: i32) -> Option<VirtualFile> {
        self.fds.remove(&fd)
    }
}

// Syscall handler example
async fn handle_openat(
    syscall: &Syscall,
    mount_table: &MountTable,
    fd_table: &mut FdTable,
) -> Result<i64> {
    let dirfd = syscall.arg0() as i32;
    let pathname_ptr = syscall.arg1() as *const c_char;
    let flags = syscall.arg2() as i32;
    let mode = syscall.arg3() as u32;

    // Read pathname from guest memory
    let pathname = unsafe { CStr::from_ptr(pathname_ptr).to_str()? };

    // Resolve path (handle AT_FDCWD, relative paths, etc.)
    let absolute_path = resolve_openat_path(dirfd, pathname, fd_table)?;

    // Find mount point
    let (mount_point, vfs, rel_path) = mount_table.find(&absolute_path)
        .ok_or_else(|| anyhow!("Path not mounted: {:?}", absolute_path))?;

    // Open file via VFS
    let file = vfs.open(rel_path.to_str().unwrap(), flags, mode).await?;

    // Allocate virtual FD
    let vfd = fd_table.insert(file);

    Ok(vfd as i64)
}
```

**When to apply this pattern**:
- ✅ Running untrusted code (user scripts, plugins)
- ✅ Testing applications in isolation
- ✅ Redirecting file I/O to custom storage (SQLite, S3, etc.)
- ✅ Linux servers or containers
- ❌ High-performance applications (10-50% overhead)
- ❌ Cross-platform requirement (ptrace is Linux-only)
- ❌ Simple use cases (FUSE or LD_PRELOAD may be simpler)

**Lessons Learned**:
- Reverie (Facebook's framework) handles ptrace complexity
- Must handle FD lifecycle carefully (leaks cause bugs)
- Static binaries work (unlike LD_PRELOAD)
- Strace mode invaluable for debugging

---

### Skill 4: Insert-Only Audit Logs

**Persona**: You are a compliance engineer or observability architect designing tamper-proof audit trails.

**When to apply**:
- Regulatory compliance (GDPR, HIPAA, SOC2)
- Security auditing (who did what when)
- Debugging distributed systems (tracing)
- Performance analysis (tool call timing)

**Questions to ask before implementing**:

1. **What events need auditing?**
   - User actions (login, data access)?
   - API calls (tool invocations)?
   - System events (errors, crashes)?

2. **What data must be captured?**
   - Timestamps (start, end, duration)?
   - Parameters (inputs to operation)?
   - Results (outputs or errors)?
   - Context (user, session, request ID)?

3. **What are the retention requirements?**
   - How long to keep audit logs?
   - Can logs be archived/compressed?
   - Who can delete logs?

4. **What are the query patterns?**
   - Time range queries (last 24 hours)?
   - Entity queries (all events for user X)?
   - Aggregations (success rate, average duration)?

**Principles**:

- **Insert-only schema**: No UPDATE or DELETE
  - Prevents tampering (immutable history)
  - Simplifies reasoning (append-only log)
  - Enables distributed replication

- **Capture timing precisely**: start, end, duration
  - Wall-clock time (Unix timestamps)
  - Duration in milliseconds (for performance analysis)
  - Computed fields (duration = end - start)

- **Store structured data as JSON**: parameters, results
  - Enables SQL queries over structured data
  - Future-proof (schema evolution via JSON)
  - Human-readable for debugging

- **Index on query hot paths**: timestamp, entity ID
  - `idx_tool_calls_started_at` for time range queries
  - `idx_tool_calls_name` for per-tool queries
  - Balance index overhead vs query speed

**Implementation Pattern**:

```sql
-- From /home/user/agentfs/SPEC.md

CREATE TABLE tool_calls (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  name TEXT NOT NULL,               -- Tool identifier
  parameters TEXT,                  -- JSON-serialized inputs
  result TEXT,                      -- JSON-serialized outputs
  error TEXT,                       -- Error message (NULL if success)
  started_at INTEGER NOT NULL,      -- Unix timestamp (seconds)
  completed_at INTEGER NOT NULL,    -- Unix timestamp (seconds)
  duration_ms INTEGER NOT NULL      -- Computed: (completed_at - started_at) * 1000
);

-- Indexes for common queries
CREATE INDEX idx_tool_calls_name ON tool_calls(name);
CREATE INDEX idx_tool_calls_started_at ON tool_calls(started_at);
```

```rust
// From /home/user/agentfs/sdk/rust/src/toolcalls.rs

pub async fn record(
    &self,
    name: &str,
    started_at: i64,
    completed_at: i64,
    parameters: Option<Value>,
    result: Option<Value>,
    error: Option<String>,
) -> Result<i64> {
    // Compute duration
    let duration_ms = (completed_at - started_at) * 1000;

    // Serialize JSON
    let params_json = parameters.map(|v| serde_json::to_string(&v)).transpose()?;
    let result_json = result.map(|v| serde_json::to_string(&v)).transpose()?;

    // Insert record (returns auto-generated ID)
    let result = self.conn.query(
        "INSERT INTO tool_calls
         (name, parameters, result, error, started_at, completed_at, duration_ms)
         VALUES (?, ?, ?, ?, ?, ?, ?)
         RETURNING id",
        (name, params_json, result_json, error, started_at, completed_at, duration_ms)
    ).await?;

    Ok(result.rows[0].get::<i64>(0)?)
}

// Query recent calls
pub async fn recent(&self, since: i64, limit: Option<usize>) -> Result<Vec<ToolCall>> {
    let sql = "SELECT * FROM tool_calls WHERE started_at > ? ORDER BY started_at DESC";
    let sql = if let Some(lim) = limit {
        format!("{} LIMIT {}", sql, lim)
    } else {
        sql.to_string()
    };

    let rows = self.conn.query(&sql, (since,)).await?;
    // ... deserialize rows
}

// Aggregate statistics
pub async fn stats(&self) -> Result<Vec<ToolCallStats>> {
    let rows = self.conn.query(
        "SELECT
           name,
           COUNT(*) as total_calls,
           SUM(CASE WHEN error IS NULL THEN 1 ELSE 0 END) as successful,
           SUM(CASE WHEN error IS NOT NULL THEN 1 ELSE 0 END) as failed,
           AVG(duration_ms) as avg_duration_ms
         FROM tool_calls
         GROUP BY name
         ORDER BY total_calls DESC",
        ()
    ).await?;
    // ... deserialize stats
}
```

**When to apply this pattern**:
- ✅ Compliance requirements (immutable audit trail)
- ✅ Security monitoring (detect anomalies)
- ✅ Performance analysis (identify slow operations)
- ✅ Debugging (trace execution flow)
- ❌ High-volume events (millions per second) without archival strategy
- ❌ Real-time alerting (query latency may be too high)

**Lessons Learned**:
- Computed fields (duration_ms) improve query performance
- JSON storage enables flexible schema evolution
- Indexes are critical for time-range queries
- Retention policy needed (logs grow unbounded)

---

## Architecture Decision Records (Inferred)

### ADR-001: Use SQLite as Foundation

**Context**:
AgentFS needed persistent storage for agent state (files, key-value data, tool calls). Requirements:
- Single-file portability
- ACID transactions
- Zero configuration
- SQL query capability
- Local and remote deployment

**Decision**: Use SQLite (via Turso libSQL fork)

**Rationale**:

**Evidence from codebase**:
- `/home/user/agentfs/sdk/rust/Cargo.toml:7` - `turso = "0.3.2"` dependency
- `/home/user/agentfs/README.md:134` - "SQLite-based storage system"
- `/home/user/agentfs/SPEC.md:1-13` - "SQLite schema" specification

**Why SQLite over alternatives**:

| Alternative | Why Rejected | Evidence |
|-------------|--------------|----------|
| PostgreSQL | Requires server, complex deployment | No postgres dependencies in Cargo.toml |
| MongoDB | No POSIX filesystem semantics | Schema uses SQL, not document model |
| Redis | Volatile storage, no SQL | No redis dependencies |
| Flat files | No transactions, no SQL queries | Schema uses tables, not files |

**Why Turso (libSQL fork)**:
- Compatible with SQLite (drop-in replacement)
- Adds remote database support (cloud deployment)
- Maintained by Turso Database company
- Same API as SQLite

**Consequences**:

**Positive**:
- ✅ Extreme simplicity (no server, no config)
- ✅ Perfect for local development (agent.db file)
- ✅ Works offline (no network required)
- ✅ Snapshot = file copy (`cp agent.db snapshot.db`)
- ✅ SQL queries for analytics

**Negative**:
- ❌ Single-writer bottleneck (SQLite limitation)
- ❌ Not ideal for multi-agent concurrent writes
- ❌ No built-in replication (mitigated by Turso remote)

**Alternatives Considered (Future)**:
- DuckDB: Better for analytics, but less portable
- libSQL extensions: Compression, replication

**Status**: ✅ Accepted and implemented

---

### ADR-002: Inode-Based Filesystem Design

**Context**:
Filesystem component needed to store files with metadata. Requirements:
- POSIX-like semantics (familiar to developers)
- Hard link support (multiple paths to same file)
- Efficient renames
- Metadata queries without reading file content

**Decision**: Use inode-based design (separate fs_inode, fs_dentry, fs_data tables)

**Rationale**:

**Evidence from codebase**:
- `/home/user/agentfs/SPEC.md:124-484` - Detailed inode filesystem specification
- `/home/user/agentfs/sdk/rust/src/filesystem.rs:7-17` - Constants for S_IFREG, S_IFDIR, etc.
- `/home/user/agentfs/sdk/rust/src/filesystem.rs:47-51` - Filesystem struct with inode operations

**Why inode design**:

1. **Hard links**: Multiple dentries point to same inode
   - Example: `/home/file.txt` and `/backup/file.txt` share inode 42
   - Deleting one doesn't delete data (link count tracks references)

2. **Efficient renames**: Update dentry, not data
   - Move `/old/path.txt` → `/new/path.txt` updates one row in fs_dentry
   - No need to rewrite file content

3. **Metadata queries**: Query fs_inode without touching fs_data
   - `SELECT size, mtime FROM fs_inode WHERE ino = ?` is fast
   - Don't need to read BLOB data for metadata

**Schema design**:
```
fs_inode:   ino → (mode, size, timestamps)  -- Metadata
fs_dentry:  (parent_ino, name) → ino        -- Namespace
fs_data:    (ino, offset) → data            -- Content
```

**Consequences**:

**Positive**:
- ✅ Full POSIX compatibility (hard links, symlinks)
- ✅ Efficient renames (O(1) instead of O(file size))
- ✅ Metadata queries fast
- ✅ Familiar to Unix developers

**Negative**:
- ❌ More complex than path-based storage
- ❌ Path resolution requires JOINs
- ❌ Need to track link counts for deletion

**Alternatives Considered**:

| Alternative | Why Rejected | Tradeoffs |
|-------------|--------------|-----------|
| Path-based (key=path, value=data) | No hard links, inefficient renames | Simpler but less POSIX-like |
| Flat storage (no directories) | No hierarchy | Too simplistic |
| MongoDB-style documents | No POSIX semantics | Wrong abstraction |

**Status**: ✅ Accepted and implemented

---

### ADR-003: Reverie Ptrace for Sandboxing

**Context**:
Sandbox component needed syscall interception to virtualize filesystem at `/agent`. Requirements:
- Intercept all filesystem syscalls
- Cannot be bypassed by user code
- Works with static binaries
- Linux-only acceptable (primary deployment target)

**Decision**: Use Facebook's Reverie ptrace framework

**Rationale**:

**Evidence from codebase**:
- `/home/user/agentfs/sandbox/Cargo.toml:11-15` - Reverie dependencies
- `/home/user/agentfs/sandbox/src/sandbox/mod.rs:1-17` - Reverie tool implementation
- `/home/user/agentfs/cli/Cargo.toml:18-21` - Reverie only on Linux

**Why Reverie**:

1. **Production-tested**: Used at Meta (Facebook) scale
   - Battle-tested on real workloads
   - Handles edge cases (multi-threading, signals, etc.)

2. **Comprehensive**: Intercepts all syscalls
   - Not just filesystem (like FUSE)
   - Can intercept network, process, etc.

3. **Type-safe**: Written in Rust
   - No manual memory management
   - Less error-prone than C ptrace

4. **Flexible**: Easy to add syscall handlers
   - `handle_syscall_event()` dispatches by syscall number
   - Can selectively intercept or pass through

**Why ptrace over alternatives**:

| Alternative | Why Rejected | Evidence |
|-------------|--------------|----------|
| FUSE | Only filesystem, not all syscalls | Need more than just filesystem |
| LD_PRELOAD | Bypassable (static binaries) | Sandbox must be secure |
| Docker/containers | Too heavy, not true isolation | Want single binary CLI |
| seccomp-bpf | Filter-only, can't virtualize | Need to redirect, not block |

**Consequences**:

**Positive**:
- ✅ Cannot be bypassed (kernel-level interception)
- ✅ Works with static binaries
- ✅ Works with any language
- ✅ Complete syscall visibility

**Negative**:
- ❌ Linux-only (ptrace is Linux kernel feature)
- ❌ ~10-50% performance overhead
- ❌ Git dependency (Reverie not on crates.io)
- ❌ Complex to debug (ptrace edge cases)

**Platform-specific compilation**:
```toml
# Only compile sandbox on Linux
[target.'cfg(target_os = "linux")'.dependencies]
agentfs-sandbox = { path = "../sandbox" }
reverie = { git = "https://github.com/facebookexperimental/reverie" }
```

**Alternatives Considered (Future)**:
- eBPF: Newer technology, but complex
- User-mode Linux: Too heavy for CLI

**Status**: ✅ Accepted and implemented (Linux-only)

---

### ADR-004: Dual-Language SDKs (Rust + TypeScript)

**Context**:
AgentFS needed to support both systems-level (CLI, sandbox) and web/Node.js (AI frameworks) use cases.

**Decision**: Maintain parallel Rust and TypeScript SDKs with identical APIs

**Rationale**:

**Evidence from codebase**:
- `/home/user/agentfs/sdk/rust/` - Full Rust implementation
- `/home/user/agentfs/sdk/typescript/` - Full TypeScript implementation
- `/home/user/agentfs/README.md:29-30` - "TypeScript and Rust libraries"
- Examples use TypeScript SDK (`examples/*/src/utils/agentfs.ts`)

**Why dual SDKs**:

1. **Ecosystem coverage**:
   - Rust: Systems programming, CLI, high-performance
   - TypeScript: Web, Node.js, AI frameworks (Claude SDK, Mastra, OpenAI)

2. **Framework compatibility**:
   - Claude Agent SDK: TypeScript only
   - Mastra: TypeScript only
   - OpenAI Agents: TypeScript only
   - Need native TypeScript integration

3. **Shared storage layer**:
   - Both SDKs read/write same SQLite schema
   - No "Rust databases" vs "TypeScript databases"
   - Enables polyglot applications

**API consistency**:
```rust
// Rust
let agentfs = AgentFS::new("agent.db").await?;
agentfs.kv.set("key", &value).await?;
```

```typescript
// TypeScript
const agentfs = await AgentFS.open('agent.db');
await agentfs.kv.set('key', value);
```

**Consequences**:

**Positive**:
- ✅ Broad developer adoption
- ✅ Works with any AI framework
- ✅ CLI (Rust) and examples (TypeScript) compatible

**Negative**:
- ❌ 2x maintenance overhead
- ❌ Risk of divergence (mitigated by tests)
- ❌ Must keep APIs in sync

**Testing strategy**:
- Cross-language compatibility tests
- Write with Rust, read with TypeScript
- Verify schema compatibility

**Alternatives Considered**:

| Alternative | Why Rejected |
|-------------|--------------|
| Rust only | Excludes TypeScript AI frameworks |
| TypeScript only | CLI would be slow, sandbox impossible |
| WASM bindings | Complex, immature tooling |
| C bindings | Unergonomic, error-prone |

**Status**: ✅ Accepted and implemented

---

## Reusable Code Patterns

### Pattern 1: Singleton Initialization with OnceLock

**Purpose**: Thread-safe lazy initialization of global state

**When to use**:
- Expensive initialization (database connections, mount tables)
- Must happen exactly once
- Accessed from multiple threads

**Implementation**:
```rust
// From /home/user/agentfs/sandbox/src/sandbox/mod.rs

use std::sync::OnceLock;

static MOUNT_TABLE: OnceLock<MountTable> = OnceLock::new();

pub fn init_mount_table(table: MountTable) {
    MOUNT_TABLE.set(table).expect("Mount table already initialized");
}

pub fn get_mount_table() -> &'static MountTable {
    MOUNT_TABLE.get().expect("Mount table not initialized")
}

// Usage
fn main() {
    let table = MountTable::new(configs);
    init_mount_table(table);  // Initialize once

    // Later, anywhere in code
    let table = get_mount_table();  // Always valid
}
```

**Why OnceLock over alternatives**:
- Lazy::new(): Requires initialization function at declaration
- Mutex<Option<T>>: Runtime overhead on every access
- Static mut: Unsafe, deprecated

---

### Pattern 2: Repository with Connection Sharing

**Purpose**: Multiple modules share single database connection

**When to use**:
- Multiple components need database access
- Want to avoid connection pool overhead
- Single-threaded or controlled concurrency

**Implementation**:
```rust
// From /home/user/agentfs/sdk/rust/src/lib.rs

use std::sync::Arc;
use turso::Connection;

pub struct AgentFS {
    conn: Arc<Connection>,  // Shared connection
    pub kv: KvStore,
    pub fs: Filesystem,
    pub tools: ToolCalls,
}

impl AgentFS {
    pub async fn new(db_path: &str) -> Result<Self> {
        let db = Builder::new_local(db_path).build().await?;
        let conn = Arc::new(db.connect()?);  // Wrap in Arc

        // Pass clones to each component
        let kv = KvStore::from_connection(conn.clone()).await?;
        let fs = Filesystem::from_connection(conn.clone()).await?;
        let tools = ToolCalls::from_connection(conn.clone()).await?;

        Ok(Self { conn, kv, fs, tools })
    }
}

// In each component
pub struct KvStore {
    conn: Arc<Connection>,  // Cheap to clone
}
```

**Benefits**:
- Single connection (no pool overhead)
- Components can't accidentally use different databases
- Connection lifetime managed by Arc

---

### Pattern 3: Path Resolution with Caching Potential

**Purpose**: Resolve filesystem paths to inodes efficiently

**When to use**:
- Hierarchical namespace (filesystem paths)
- Lookups dominate workload
- Caching can improve performance

**Implementation**:
```rust
// From /home/user/agentfs/sdk/rust/src/filesystem.rs

async fn resolve_path(&self, path: &str) -> Result<Option<i64>> {
    if path == "/" {
        return Ok(Some(ROOT_INO));  // Fast path
    }

    let mut current_ino = ROOT_INO;
    for component in path.split('/').filter(|s| !s.is_empty()) {
        // TODO: Cache (component, parent_ino) → child_ino
        let result = self.conn.query(
            "SELECT ino FROM fs_dentry WHERE parent_ino = ? AND name = ?",
            (current_ino, component)
        ).await?;

        if result.rows.is_empty() {
            return Ok(None);
        }

        current_ino = result.rows[0].get::<i64>(0)?;
    }
    Ok(Some(current_ino))
}
```

**Optimization opportunity**:
```rust
// Add LRU cache
use lru::LruCache;

struct Filesystem {
    conn: Arc<Connection>,
    path_cache: Mutex<LruCache<String, i64>>,  // path → ino
}

async fn resolve_path(&self, path: &str) -> Result<Option<i64>> {
    // Check cache first
    if let Some(ino) = self.path_cache.lock().unwrap().get(path) {
        return Ok(Some(*ino));
    }

    // ... expensive resolution ...

    // Cache result
    if let Some(ino) = ino {
        self.path_cache.lock().unwrap().put(path.to_string(), ino);
    }

    Ok(ino)
}
```

---

## Lessons Learned

### 1. SQLite is Powerful but Has Limits

**What worked**:
- Single-file portability is magical for users
- ACID transactions prevent corruption
- SQL queries enable powerful analytics

**What didn't**:
- Single-writer limits multi-agent concurrency
- Large BLOBs (>10MB files) can be slow
- Need archival strategy (audit logs grow unbounded)

**Recommendation**:
- Perfect for local development and single-agent use
- For multi-agent: Use separate databases per agent
- For large files: Consider chunking or external storage

---

### 2. Dual-Language SDKs are Worth the Cost

**What worked**:
- TypeScript SDK enabled all AI framework examples
- Rust SDK enabled performant CLI and sandbox
- Shared storage prevented fragmentation

**What didn't**:
- Keeping APIs in sync requires discipline
- Testing cross-language compatibility is complex
- Documentation must be language-neutral

**Recommendation**:
- Start with one language, prove the design
- Port to second language with automated tests
- Use integration tests (write Rust, read TypeScript)

---

### 3. Ptrace Sandboxing is Complex but Powerful

**What worked**:
- Reverie handles most ptrace complexity
- Cannot be bypassed (secure isolation)
- Works with static binaries

**What didn't**:
- Linux-only limits portability
- Performance overhead (~10-50%)
- Debugging is difficult

**Recommendation**:
- Use Reverie (don't write raw ptrace)
- Add strace mode for debugging
- Accept Linux-only constraint for security

---

### 4. Insert-Only Audit Logs are Simple and Effective

**What worked**:
- Immutable history prevents tampering
- SQL aggregations enable analytics
- JSON storage allows schema evolution

**What didn't**:
- Unbounded growth (need archival)
- Query performance degrades with size

**Recommendation**:
- Perfect for compliance and debugging
- Add retention policy (archive old records)
- Index on query hot paths

---

## When to Use AgentFS Patterns

| Pattern | Use When | Avoid When |
|---------|----------|------------|
| SQLite-as-Filesystem | Single-file portability needed, POSIX semantics | High concurrency, streaming large files |
| Dual-Language SDKs | Multiple ecosystems, shared storage | One language dominates, rapid iteration |
| Syscall Interception | Untrusted code, custom storage, Linux-only OK | Cross-platform, high performance critical |
| Insert-Only Audit Logs | Compliance, debugging, immutability | High-volume events, real-time alerting |
| Inode Design | Hard links, efficient renames, POSIX | Simple use case, no hard links needed |

---

## Future Research Directions

Based on gaps identified:

1. **Multi-agent Coordination**: Distributed locking for multiple agents sharing database
2. **Compression**: Transparent compression for large files (zstd, lz4)
3. **Encryption**: SQLite encryption extension for data at rest
4. **Quota Management**: Track and enforce storage limits per agent
5. **Event Sourcing**: Record all filesystem events for time-travel debugging
6. **Cross-platform Sandbox**: WASM or other cross-platform isolation

---

**Conclusion**: AgentFS demonstrates sophisticated patterns for building portable, auditable storage for AI agents. The skills extracted here are reusable for similar projects requiring SQLite-based filesystems, dual-language SDKs, or Linux sandboxing.
