We are planning to develop Column Family concept for RocksDB. Each key-value pair will be assigned to a Column Family and will be logically separated from key-value pairs in other Column Families. Here are some of the properties we are proposing:
* We will support atomic writes across Column Families. For example, a user will be able to atomically execute Write([(cf1, key1, value1), (cf2, key2, value2)]). 
* A user can request consistent state of the database across Column Families using MultiGet() or NewIterator() calls.
* We won't support iterations/scans across Column Families. We will still support iterations within one Column Family.
* Each Column Family will have its own options, which will enable user to tune performance, compression and compaction strategy for each Column Family.
* We will support on-the-fly adding and deleting Column Families.
* Our goal is to be able to have [a big number] of Column Families per RocksDB instance without big performance hits.

### Implementation details:
* There will be only one transaction log and manifest file per RocksDB instance. However, each Column Family will have its own memtables and sst files.
* Memtables will be flushed on their own schedule. When one Column Family schedules a memtable flush, we will try to also flush as many of other Column Families as possible. However, if a certain Column Family is just not ready for a memtable flush, we will not touch it.
* After some memtable flushes (not on every flush), we will also roll the transaction log. We will create a new transaction log and put all new writes to that log. Some live memtables, however, might still have data in the old log. We will not delete old logs until all memtables that have data in them get flushed. In case data in logs grows big, we will have an option to trigger flushes for memtables that have data in them. Once memtable flushes are done, we will be able to delete old logs and free the space.
* All the compactions will be executed on a single Column Family, configured with its own specific options.
* Manifest file will contain the Column Family metadata. This includes the name of the Column Family and associated sst tables (and their metadata). When recovering, we need to be careful not to replay the updates in the log that are already persistent in the sst files - RocksDB updates are not idempotent because of a Merge Operator!
* Since we're changing the Manifest file format, we will read the old Manifest file format and write out a new format during DB::Open(). We support backwards compatibility, but not forward compatibility - databases created with new versions of RocksDB will not be readable by older versions of RocksDB. All key-values without any Column Family specified will be stored in Column Family "default"
* SST tables will also include Column Family name in the footer. That way, we will be able to recover a database even if we are missing a Manifest file.
* Replication log format will now include column family reference with every key-value.
* We will still use a single DB mutex to lock the database's metadata.
* When a Column Family gets deleted, we will first atomically update Manifest to record the deletion. Then, we can either delete all the Column Family's data files or return immediately and delete the data lazily during the compaction process in the background thread.

### API
* On DB::Open, client must pass in a vector of <column_family_name, ColumnFamilyOptions> that will describe the column families.
* Client is responsible for persisting vector<ColumnFamilyOptions> across DB runtimes. DB Open will fail if a client's specified column families are not found in Manifest, or if some column families specified in Manifest are not configured.
* We will add DB::AddColumnFamily(column_family_name, ColumnFamilyOptions), DB::DropColumnFamily(column_family_name) and DB::ListColumnFamilies().
* All data access/storage/scan APIs will be changed to include Slice column_family. The old APIs will call the new API with default column family (empty string)
* NewIterator call with a sequence number will enable user to get consistent view of the database across column families. 
* Currently, Options class includes options that are specific to certain column family (write_buffer_size, compression, etc.) as well as some DB-specific options (create_if_not_exists, wal_dir, etc.) We will need to break that out to DBOptions and ColumnFamilyOptions. Here are some options that will become ColumnFamilyOptions: env (can be shared across column family or not), comparator, merge_operator, compaction_filter, max_write_buffer_size, min_write_buffer_number_to_merge, block_size, block_restart_interval, compression, compression_per_level, filter_policy, whole_key_filtering, etc. To stay backwards compatible, we will leave those options in Options class as well. They will specify ColumnFamilyOptions for default Column Family "default".