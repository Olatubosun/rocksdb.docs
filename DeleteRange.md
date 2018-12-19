# Overview

DeleteRange is an operation designed to replace the following pattern where a user wants to delete a range of keys in the range `[start, end)`:

```c++
...
Slice start, end;
// set start and end
auto it = db->NewIterator(ReadOptions());

for (it->Seek(start); cmp->Compare(it->key(), end) < 0; it->Next()) {
  db->Delete(WriteOptions(), it->key());
}
...
```
 
This pattern requires performing a range scan, which prevents it from being an atomic operation, and makes it unsuitable for any performance-sensitive write path. To mitigate this, RocksDB provides a native operation to perform this task:
```c++
...
Slice start, end;
// set start and end
db->DeleteRange(WriteOptions(), start, end);
...
```

Under the hood, this creates a range tombstone represented as a single kv, which significantly speeds up write performance. Read performance with range tombstones is competitive to the scan-and-delete pattern. (For a more detailed performance analysis, see [the DeleteRange blog post](https://rocksdb.org/blog/2018/11/21/delete-range.html).

# Internals [WIP]

The rest of this document is aimed at engineers familiar with the RocksDB codebase and conventions.

## Tombstone Fragments

The DeleteRange API does not provide any restrictions on the ranges it can delete (though if `start >= end`, the deleted range will be considered empty). This means that ranges can overlap and cover wildly different numbers of keys. The lack of structure in the range tombstones created by a user make it impossible to binary search range tombstones. For example, suppose a DB contained the range tombstones `[c, d)@4`, `[g, h)@7`, and `[a, z)@10`, and we were looking up the key `e@1`. If we sorted the tombstones by start key, then we would select `[c, d)`, which doesn't cover `e`. If we sorted the tombstones by end key, then we would select `[g, h)`, which doesn't cover `e`. However, we see that `[a, z)` does cover `e`, so in both cases we've looked at the wrong tombstone. If we left the tombstones in this form, we would have to scan through all of them in order to find a potentially covering tombstone. This linear search overhead on the read path becomes very costly as the number of range tombstones grows.

However, it is possible to transform these tombstones using the following insight: one range tombstone is equivalent to several contiguous range tombstones at the same sequence number; for example, `[a, d)@4` is equivalent to `[a, b)@4; [b, d)@4`. Using this fact, we can always "fragment" a list of range tombstones into an equivalent set of tombstones that are non-overlapping. From the example earlier, we can convert `[c, d)@4`, `[g, h)@7`, and `[a, z)@10` into: `[a, c)@10`, `[c, d)@10`, `[c, d)@4`, `[d, g)@10`, `[g, h)@10`, `[g, h)@7`, `[h, z)@10`.

Now that the tombstone fragments do not overlap, we can safely perform a binary search. Going back to the `e@1` example, binary search would find `[d, g)@10`, which covers `e@1`, thereby giving the correct result.

### Fragmentation Algorithm

See [db/range_tombstone_fragmenter.cc](https://github.com/facebook/rocksdb/blob/master/db/range_tombstone_fragmenter.cc).

## Write Path

### Memtable

When `DeleteRange` is called, a kv is written to a dedicated memtable for range tombstones. The format is `start : end` (i.e., `start` is the key, `end` is the value). Range tombstones are not fragmented in the memtable directly, but are instead fragmented each time a read occurs.

See [Compaction](#compaction) for details on how range tombstones are flushed to SSTs.

### SST Files

Just like in memtables, range tombstones in SSTs are not stored inline with point keys. Instead, they are stored in a dedicated meta-block for range tombstones. Unlike in memtables, however, range tombstones are fragmented and cached along with the table reader when the table reader is first created. Each SST file contains all range tombstones at that level that cover user keys overlapping with the file's key range; this greatly simplifies iterator seeking and point lookups.

See [Compaction](#compaction) for details on how range tombstones are compacted down the LSM tree.

## Read Path

Due to the simplicity of the write path, the read path requires more work.

### Point Lookups

When a user calls `Get` and the point lookup progresses down the LSM tree, the range tombstones in the table being searched are first fragmented (if not fragmented already) and binary searched before checking the file's contents. This is done through `FragmentedRangeTombstoneIterator::MaxCoveringTombstoneSeqnum` (see [db/range_tombstone_fragmenter.cc](https://github.com/facebook/rocksdb/blob/master/db/range_tombstone_fragmenter.cc)); if a tombstone is found (i.e., the return value is non-zero), then we know that there is no need to search lower levels since their merge operands / point values are deleted. We check the current level for any keys potentially written after the tombstone fragment, process the merge operands (if any), and return. This is similar to how finding a point tombstone stops the progression of a point lookup.

### Range Scans

The key idea for reading range deletions during point scans is to create a structure resembling the merging iterator used by `DBIter` for point keys, but for the range tombstones in the set of tables being examined.

Here is an example to illustrate how this works for forward scans (reverse scans are similar):

Consider a DB with a memtable containing the tombstone fragment `[a, b)@40, [a, b)@35`, and 2 L0 files `1.sst` and `2.sst`. `1.sst` contains the tombstone fragments `[a, c)@15, [d, f)@20`, and `2.sst` contains the tombstone fragments `[b, e)@5, [e, x)@10`. Aside from the merging iterator of point keys in the memtable and SST files, we also keep track of the following 3 data structures:
1. a min-heap of fragmented tombstone iterators (one iterator per table) ordered by end key (*active heap*)
2. an ordered set of tombstones ordered by sequence number (*active seqnum set*)
3. a min-heap of tombstones ordered by start key (*inactive heap*)

The active heap contains all iterators pointing at tombstone fragments that cover the most recent (internal) lookup key, the active seqnum set contains the seqnums of the iterators that are in the active heap, and the inactive heap contains iterators pointing at tombstone fragments that start after the most recent lookup key. Note that an iterator is not allowed to be in both the active and inactive set.

Suppose the internal merging iterator in `DBIter` points to the internal key `a@4`. The active iterators would be the tombstone iterator for `1.sst` pointing at `[a, c)@15` and the tombstone iterator for the memtable pointing at `[a, b)@40`, and the only inactive iterator would be the tombstone iterator for `2.sst` pointing at `[b, e)@5`. The active seqnum set would contain `{40, 15}`. From these data structures, we know that the largest covering tombstone has a seqnum of 40, which is larger than 4; hence, `a@4` is deleted and we need to check another key.

Next, suppose we then check `b@50`. In response, the active heap now contains the iterators for `1.sst` pointing at `[a, c)@15` and `2.sst` pointing at `[b, e)@5`, the inactive heap contains nothing, and the active seqnum set contains `{15, 5}`. Note that the memtable iterator is now out of scope and is not tracked by these data structures. (Even though the memtable iterator had another tombstone fragment, the fragment was identical to the previous one except for its seqnum (which is smaller), so it was skipped.) Since the largest seqnum is 15, `b@50` is not covered.

For implementation details, see [db/range_del_aggregator.cc](https://github.com/facebook/rocksdb/blob/master/db/range_del_aggregator.cc).

## Compaction

# Future Work [WIP]
[TODO: This section should really be moved to an issue]

- tombstone iterator lifetime management
- memtable caching
- snapshot-release compactions
- range tombstone-aware compaction scheduling
- range tombstone-aware file boundaries (based on grandparent SSTs)
- new format version proposal
- transaction support