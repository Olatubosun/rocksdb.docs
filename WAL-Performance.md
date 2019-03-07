# Non-Sync Mode
When `WriteOptions.sync = false` (the default), WAL writes are not synchronized to disk. Unless the operating system thinks it must flush the data (e.g. too many dirty pages), users don't need to wait for any I/O for write.

Users who want to even reduce the CPU of latency introduced by writing to OS page cache, can choose `Options.manual_wal_flush = true`. With this option, WAL writes are not even flushed to the file system page cache, but kept in RocksDB. Users need to call `DB::FlushWAL()` to have buffered entries go to the file system.

Users can call DB::SyncWAL() to force fsync WAL files. The function will not block writes being executed in other threads.

In this mod, the WAL write is not crash safe.

# Sync Mode
When `WriteOptions.sync = true`, the WAL file is fsync'ed before returning to the user.

## Group Commit
As most other systems relying on logs, RocksDB supports **group commit** to improve WAL writing throughput, as well as write amplification. RocksDB's group commit is implemented in a naive way: when different threads are writing to the same DB at the same time, all outstanding writes that qualify to be combined will be combined together and write to WAL once, with one fsync. In this way, more writes can be completed by the same number of I/Os.

Writes with different write options might disqualify themselves to be combined. The maximum group size is 1MB. RocksDB won't try to increase batch size by proactive delaying the writes.

