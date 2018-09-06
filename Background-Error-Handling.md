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

Once the database instance goes into read-only mode, the following foreground operations will return the error status on all subsequent calls -
Write
Put
IngestExternalFile
CompactFiles
CompactRange
Flush

In addition to the above, a notification callback ```EventListener::OnBackgroundError``` will be called as soon as the background error is encountered.

There are 2 possible ways to handle a background error -
1. The ```EventListener::OnBackgroundError``` callback can override the error status if it determines that its not serious enough to stop further writes to the DB. It can do so by setting the ```bg_error``` parameter. Doing so can be risky, as RocksDB may not be able to guarantee the consistency of the DB. Check the ```BackgorundErrorReason``` and severity of the error before overriding it.
2. Call ```DB::Resume()``` to manually resume the DB and put it in read-write mode. This function will clear the error, purge any obsolete files, and restart background flush and compaction operations. At present, it only supports resuming from background errors that happen during compaction. In the future, we will add more cases.