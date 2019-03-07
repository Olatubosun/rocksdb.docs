# Non-Sync Mode
When `WriteOptions.sync = false` (the default), WAL writes are not synchronized to disk. Unless the operating system thinks it must flush the data (e.g. too many dirty pages), users don't need to wait for any I/O for write.

Users who want to even reduce the CPU of latency introduced by writing to OS page cache, can choose `Options.manual_wal_flush = true`. With this option, WAL writes are not even flushed to the file system page cache, but kept in RocksDB. Users need to call `DB::FlushWAL()` to have buffered entries go to the file system.

In this mod, the WAL write is not crash safe.

# Sync Mode
When `WriteOptions.sync = true`, the WAL file is fsync'ed before returning to the user.