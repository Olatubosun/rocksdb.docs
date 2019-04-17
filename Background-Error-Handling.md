Currently in RocksDB, any error during a write operation (write to WAL, Memtable Flush, background compaction etc) causes the database instance to go into read-only mode by default and further user writes are not accepted. The variable ```ErrorHandler::bg_error_``` is set to the failure ```Status```. The ```DBOptions::paranoid_checks``` option controls how aggressively RocksDB will check and flag errors. The default value of this option is true. The table below shows the various scenarios where errors are considered and potentially affect database operations.

| Error Reason | When bg_error_ is set |
-----------------|-----------------------
| Sync WAL (```BackgroundErrorReason::kWriteCallback```) | Always |
| Memtable insert failure (```BackgroundErrorReason::kMemTable```) | Always |
| Memtable flush (```BackgroundErrorReason::kFlush```) | DBOptions::paranoid_checks is true |
| ```SstFileManager::IsMaxAllowedSpaceReached()``` reports max allowed space reached during memtable flush (```BackgroundErrorReason::kFlush```) |	Always |
| ```SstFileManager::IsMaxAllowedSpaceReached()``` reports max allowed space reached during compaction (```BackgroundErrorReason::kCompaction```) | Always |
| ```DB::CompactFiles``` (```BackgroundErrorReason::kCompaction```) | DBOptions::paranoid_checks is true |
| Background compaction (```BackgroundErrorReason:::kCompaction```) | DBOptions::paranoid_checks is true |
| Write (```BackgroundErrorReason::kWriteCallback```) | DBOptions::paranoid_checks is true |

# Detection
Once the database instance goes into read-only mode, the following foreground operations will return the error status on all subsequent calls -
* ```DB::Write```, ```DB::Put```, ```DB::Delete```, ```DB::SingleDelete```, ```DB::DeleteRange```, ```DB::Merge```
* ```DB::IngestExternalFile```
* ```DB::CompactFiles```
* ```DB::CompactRange```
* ```DB::Flush```

The returned ```Status``` will indicate the error code, sub-code as well as severity. The severity of the error can be determined by calling ```Status::severity()```. There are 4 severity levels and they are defined as follows -
1. ```Status::Severity::kSoftError``` - Errors of this severity do not prevent writes to the DB, but it does mean that the DB is in a degraded mode. Background compactions may not be able to run in a timely manner.
2. ```Status::Severity::kHardError``` - The DB is in read-only mode, but it can be transitioned back to read-write mode once the cause of the error has been addressed.
3. ```Status::Severity::kFatalError``` - The DB is in read-only mode. The only way to recover is to close the DB, remedy the underlying cause of the error, and then re-open the DB.
4. ```Status::Severity::kUnrecoverableError``` - This is the highest severity and indicates a corruption in the database. It may be possible to close and re-open the DB, but the contents of the database may no longer be correct.  

In addition to the above, a notification callback ```EventListener::OnBackgroundError``` will be called as soon as the background error is encountered.

# Recovery
There are 2 possible ways to recover from a background error without shutting down the database -
1. The ```EventListener::OnBackgroundError``` callback can override the error status if it determines that its not serious enough to stop further writes to the DB. It can do so by setting the ```bg_error``` parameter. Doing so can be risky, as RocksDB may not be able to guarantee the consistency of the DB. Check the ```BackgorundErrorReason``` and severity of the error before overriding it.
2. Call ```DB::Resume()``` to manually resume the DB and put it in read-write mode. This function will flush memtables for all the column families, clear the error, purge any obsolete files, and restart background flush and compaction operations. 
3. Automatic recovery from background errors. This is done by polling the system to ensure the underlying error condition is resolved, and then following the same steps as ```DB::Resume()``` to restore write capability. Notification callbacks ```EventListener::OnErrorRecoveryBegin``` and ```EventListener::OnErrorRecoveryCompleted``` are called at the start and end of the recovery process respectively, to inform the user of the status. At present, the automatic recovery is only supported for ENOSPC error from the filesystem. If the error happens during WAL sync, recovery is done only if 2PC is not in use. In the future, we will add more cases.