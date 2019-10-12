MemTable is an in-memory data-structure holding data before they are flushed to SST files. It serves both read and write - new writes always insert data to memtable, and reads has to query memtable before reading from SST files, because data in memtable is newer. Once a memtable is full, it becomes immutable and replace by a new memtable. A background thread will flush the content of the memtable into a SST file, after which the memtable can be destroyed.

The most important options that affect memtable behavior are:

* `memtable_factory`: The factory object of memtable. By specifying factory object user can change the underlying implementation of memtable, and provide implementation specific options.
* `write_buffer_size`: Size of a single memtable.
* `db_write_buffer_size`: Total size of memtables across column families. This can be used to manage the total memory used by memtables.
* `write_buffer_manager`: Instead of specifying a total size of memtables, user can provide their own write buffer manager to control the overall memtable memory usage. Overrides `db_write_buffer_size`.
* `max_write_buffer_number`: The maximum number of memtables build up in memory, before they flush to SST files.

The default implementation of memtable is based on skiplist. Other than the default memtable implementation, users can use other types of memtable implementation, for example HashLinkList, HashSkipList or Vector, to speed-up some queries.

### Skiplist MemTable

Skiplist-based memtable provides general good performance to both read and write, random access and sequential scan. Besides, it provides some other useful features that other memtable implementations don't currently support, like [[Concurrent Insert|memtable#concurrent-insert]] and [[Insert with Hint|memtable#insert-with-hint]].

### HashSkiplist MemTable

As their names imply, HashSkipList organizes data in a hash table with each hash bucket to be a skip list, while HashLinkList organizes data in a hash table with each hash bucket as a sorted single linked list. Both types are built to reduce number of comparisons when doing queries. One good use case is to combine them with PlainTable SST format and store data in RAMFS.

When doing a look-up or inserting a key, target key's prefix is retrieved using Options.prefix_extractor, which is used to find the hash bucket. Inside a hash bucket, all the comparisons are done using whole (internal) keys, just as SkipList based memtable.

The biggest limitation of the hash based memtables is that doing scan across multiple prefixes requires copy and sort, which is very slow and memory costly.

### Flush

There are three scenarios where memtable flush can be triggered:

1. Memtable size exceeds `write_buffer_size` after a write.
2. Total memtable size across all column families exceeds `db_write_buffer_size`, or `write_buffer_manager` signals a flush. In this scenario the largest memtable will be flushed.
3. Total WAL file size exceeds `max_total_wal_size`. In this scenario the memtable with the oldest data will be flushed, in order to allow the WAL file with data from this memtable to be purged.

As a result, a memtable can be flushed before it is full. This is one reason the generated SST file can be smaller than the corresponding memtable. Compression is another factor to make SST file smaller than corresponding memtable, since data in memtable is uncompressed.

### Concurrent Insert

Without support of concurrent insert to memtables, concurrent writes to RocksDB from multiple threads will apply to memtable sequentially. Concurrent memtable insert is enabled by default and can be turn off via `allow_concurrent_memtable_write` option, although only skiplist-based memtable supports the feature.

### Insert with Hint

### In-place Update

### Comparison

| Mem Table Type | SkipList | HashSkipList | HashLinkList | Vector |
|----------------|----------|--------------|--------------|--------|
| Optimized Use Case                    | General                               | Range query within a specific key prefix                                                     | Range query within a specific key prefix and there are only a small number of rows for each prefix | Random write heavy workload |
| Index type                            | binary search                         | hash + binary search                                                                         | hash + linear search                                                                               | linear search |
| Support totally ordered full db scan? | naturally                             | very costly (copy and sort to create a temporary totally-ordered view)                       | very costly (copy and sort to create a temporary totally-ordered view)                             | very costly (copy and sort to create a emporary totally-ordered view) |
| Memory Overhead                       | Average (multiple pointers per entry) | High (Hash Buckets + Skip List Metadata for non-empty buckets + multiple pointers per entry) | Lower (Hash buckets + pointer per entry)                                                           | Low (pre-allocated space at the end of vector) |
| MemTable Flush                        | Fast with constant extra memory       | Slow with high temporary memory usage                                                        | Slow with high temporary memory usage                                                              | Slow with constant extra memory |
| Concurrent Insert | Support | Not support | Not support | Not support|
| Insert with Hint | Support (in case there are no concurrent insert) | Not support | Not support | Not support |

