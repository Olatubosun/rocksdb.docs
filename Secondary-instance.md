RocksDB supports shared access to a database directory in a primary-secondary mode. The primary instance is a regular RocksDB instance capable of read, write, flush and compaction. (Or just compaction since flush can be viewed as a special type of compaction of L0 files). The secondary instance is similar to read-only instance, since it supports read but not write, flush or compaction. However, the secondary instance is able to dynamically tail the MANIFEST and write-ahead-logs (WALs) of the primary and apply the related changes if applicable. User has to call `DB::TryCatchUpWithPrimary()` explicitly at chosen times.

Example usage
```
const std::string kDbPath = "/tmp/rocksdbtest";
...
// Assume we have already opened a regular RocksDB instance db_primary
// whose database directory is kDbPath.
assert(db_primary);
Options options;
options.max_open_files = -1;
// Secondary instance needs its own directory to store info logs (LOG)
const std::string kSecondaryPath = "/tmp/rocksdb_secondary/";
DB* db_secondary = nullptr;
Status s = DB::OpenAsSecondary(options, kDbPath, kSecondaryPath, &db_secondary);
assert(!s.ok() || db_secondary);
// Let secondary **try** to catch up with primary
s = db_secondary->TryCatchUpWithPrimary();
assert(s.ok());
// Read operations
std::string value;
s = db_secondary->Get(ReadOptions(), "foo", &value);
...
```

Current Limitations and Caveats
- The secondary instance must be opened with `max_open_files = -1`, indicating the secondary has to keep all file descriptors open in order to prevent them from becoming inaccessible after the primary unlinks them, which does not work on some non-POSIX file systems. We have a plan to relax this limitation in the future.
- RocksDB relies heavily on compaction to improve read performance. If a secondary instance tails and applies the log files of the primary right before a compaction, then it is possible to observe worse read performance on the secondary instance than on the primary until the secondary tails the logs again and advances to the state after compaction.