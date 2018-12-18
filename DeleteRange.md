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
 
This pattern requires performing a range scan, which prevents it from being usable on any performance-sensitive write path. To mitigate this, RocksDB provides a native operation to perform this task:
```c++
...
Slice start, end;
// set start and end
db->DeleteRange(WriteOptions(), start, end);
...
```

Under the hood, this creates a range tombstone represented as a single kv, which significantly speeds up write performance. Read performance with range tombstones is competitive to the scan-and-delete pattern. (For a more detailed performance analysis, see [the DeleteRange blog post](https://rocksdb.org/blog/2018/11/21/delete-range.html).

# Internals [WIP]

## Tombstone Fragments

The DeleteRange API does not provide any restrictions on the ranges it can delete (though if `start >= end`, the deleted range will be considered empty). This means that ranges can overlap and cover wildly different numbers of keys. The lack of structure in the range tombstones created by a user make it impossible to binary search range tombstones. For example, suppose a DB contained the range tombstones `[c, d)@4`, `[g, h)@7`, and `[a, z)@10`, and we were looking up the key `e@1`. If we sorted the tombstones by start key, then we would select `[c, d)`, which doesn't cover `e`. If we sorted the tombstones by end key, then we would select `[g, h)`, which doesn't cover `e`. However, we see that `[a, z)` does cover `e`, so in both cases we've looked at the wrong tombstone. If we left the tombstones in this form, we would have to scan through all of them in order to find a potentially covering tombstone. This linear search overhead on the read path becomes very costly as the number of range tombstones grows.

However, it is possible to transform these tombstones using the following insight: one range tombstone is equivalent to several contiguous range tombstones at the same sequence number; for example, `[a, d)@4` is equivalent to `[a, b)@4; [b, d)@4`. Using this fact, we can always "fragment" a list of range tombstones into an equivalent set of tombstones that are non-overlapping. From the example earlier, we can convert `[c, d)@4`, `[g, h)@7`, and `[a, z)@10` into: `[a, c)@10`, `[c, d)@10`, `[c, d)@4`, `[d, g)@10`, `[g, h)@10`, `[g, h)@7`, `[h, z)@10`.

Now that the tombstone fragments do not overlap, we can safely perform a binary search. Going back to the `e@1` example, binary search would find `[d, g)@10`, which covers `e@1`, thereby giving the correct result.

### Fragmentation Algorithm

See [db/range_tombstone_fragmenter.cc](https://github.com/facebook/rocksdb/blob/master/db/range_tombstone_fragmenter.cc).

## Write Path
### Memtable
### SST Files

## Read Path
### point lookups
### range scans (talk about tombstone truncation)
### Reads during compaction

# Future Work [WIP]

- tombstone iterator lifetime management
- memtable caching
- snapshot-release compactions
- new format version proposal