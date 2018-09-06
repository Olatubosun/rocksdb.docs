Currently in RocksDB, any error during a write operation (write to WAL, Memtable Flush, background compaction etc) causes the database instance to go into read-only mode by default and further user writes are not accepted. The ```DBOptions::paranoid_checks``` option controls how aggressively RocksDB will check and flag errors. The default value of this option is true. The table below shows the various scenarios where errors are considered and potentially affect database operations.

Error Reason	When bg_error_ is set
Sync closed logs	Always
Memtable flush	DBOptions::paranoid_checks is true
SFM max allowed space reached during memtable flush	Always
SFM max allowed space reached during compaction	Always
Compact Files	DBOptions::paranoid_checks is true
Background compaction	DBOptions::paranoid_checks is true
Write	DBOptions::paranoid_checks is true
