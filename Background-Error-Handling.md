Currently in RocksDB, any error during a write operation (write to WAL, Memtable Flush, background compaction etc) causes the database instance to go into read-only mode by default and further user writes are not accepted. The variable ```ErrorHandler::bg_error_``` is set to the failure ```Status```. The ```DBOptions::paranoid_checks``` option controls how aggressively RocksDB will check and flag errors. The default value of this option is true. The table below shows the various scenarios where errors are considered and potentially affect database operations.

| Error Reason | When bg_error_ is set |
-----------------|-----------------------
| Sync WAL | Always |
| Memtable flush | DBOptions::paranoid_checks is true |
| Max allowed space reached during memtable flush (```SstFileManager::IsMaxAllowedSpaceReached()```) |	Always |
| Max allowed space reached during compaction (```SstFileManager::IsMaxAllowedSpaceReached()```) | Always |
| ```DB::CompactFiles``` | DBOptions::paranoid_checks is true |
| Background compaction | DBOptions::paranoid_checks is true |
| Write | DBOptions::paranoid_checks is true |
