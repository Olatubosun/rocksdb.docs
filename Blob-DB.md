BlobDB is a RocksDB wrapper that provides key-value separation. Large values (blobs) are stored in separate blob log files, and the blob index is stored in RocksDB's LSM tree. The blob index comprises of the key, along with blob-metadata as value. This is inspired by the [WiscKey](https://www.usenix.org/system/files/conference/fast16/fast16-papers-lu.pdf) paper. By separating value storage from the LSM tree, BlobDB provides an alternative way to reduce write amplification, instead of tuning compactions. BlobDB is used in production at Facebook.

### API
The API is defined in "rocksdb/utilities/blob_db/blob_db.h" header file. (Note that this file is not part of the "include" directory i.e not exposed in the public API yet).

### Implementation
BlobDB is implemented as a wrapper on top of RocksDB, using the StackableDB interface. On a write, BlobDB first appends the blob to blob file and obtains the blob offset, then construct the blob index and writes to base RocksDB. A compaction filter is used to remove expired blobs on compaction. Blob indexes of overwritten keys will be compacted out as normal values.

### Where could this be useful?
BlobDB currently doesn't support some RocksDB features, like merge operator and column families.
In its current state, BlobDB is good for use cases that don't require generic GC and can tolerate some data loss (not using WAL), and which use TTL or FIFO eviction instead of GC. 

**FIFO use cases**

FIFO compaction is usually selected to reduce the flash burn rate for data that usually has a TTL. These use cases are frequently a good fit for blob storage. Some of these use cases may not be satisfied with what FIFO provides. BlobDB may provide value for current FIFO users if:

* They want better read performance while keeping the goodness of FIFO.
* They want per-key TTL instead of per-table TTL.

### Limitations
There are some RocksDB features which are not supported yet in BlobDB, but we are working on them. Some missing features (not an exhaustive list though):
- Merge, SingleDelete, RangeDelete APIs
- User Compaction filters
- Transactions
- A good garbage collector

### Tooling
`blob_dump` tool can be used to dump the contents of the blob files. It can be built using `make blob_dump`.