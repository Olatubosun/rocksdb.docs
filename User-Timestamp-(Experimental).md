**This feature is in early, experimental stage. Both API and internal implementation are subject to change.**

We plan to allow users to assign timestamps to their data. Users can choose the format/length/encoding of the timestamp, they just need to tell RocksDB how to parse. In order to use the feature, user has to pass a custom `Comparator` as an option to the database during open. Once opened, the timestamp format is fixed unless a full compaction is run. Within one DB, the length of timestamp has to be fixed.

In the current implementation, user-specified timestamp is stored between original user key and RocksDB internal sequence number, as follows.
```
|user key|timestamp|seqno|type|
|<-------internal key-------->|
```

# API
At present, `Get()`, `Put()`, and `Write()` are supported. Details of the API can be found in db.h. This list is not complete, and we plan to add more API functions in the future.

## Open DB
```
class MyComparator : public Comparator {
 public:
  // Compare lhs and rhs, taking timestamp, if exists, into consideration.
  int Compare(const Slice& lhs, const Slice& rhs) const override {...}
  // Compare two timestamps ts1 and ts2.
  int CompareTimestamp(const Slice& ts1, const Slice& ts2) const override {...}
  // Compare a and b after stripping timestamp from them.
  int CompareWithoutTimestamp(const Slice& a, const Slice& b) const override {...}
};

int main() {
  MyComparator cmp;
  Options options;
  options.comparator = &cmp;
  // Set other fields of options and open DB
  ...
  return 0;
}
```
## Get
`Get()` with a timestamp `ts` specified in `ReadOptions` will return the most recent key/value whose timestamp is smaller than or equal to `ts`. Note that if the database enables timestamp, then caller of `Get()` 
should set `ReadOptions.timestamp`.
```
ReadOptions read_opts;
read_opts.timestamp = &ts;
s = db->Get(read_opts, key, &value);
``` 
## Put
When using `Put` with user timestamp, the user needs to set `WriteOptions.timestamp`.
```
WriteOptions write_opts;
write_opts.timestamp  = &ts;
s = db->Put(write_opts, key, value);
```
## Write
When using `Write` API, the user creates a `WriteBatch`. The user does not need to set `WriteOptions.timestamp`. Instead, the user specifies timestamp size when creating `WriteBatch`, and calls `WriteBatch::AssignTimestamp(...)` before calling `Write`. `WriteBatch::AssignTimestamp()` can assign one or multiple timestamps to the key/value pairs in the write batch.
```
const size_t timestamp_size = 8; // 8-byte timestamp
WriteBatch wb(reserved_bytes, max_bytes, timestamp_size);
wb.Put(key1, value1);
wb.Delete(key2);
wb.AssignTimestamp(ts);
s = db->Write(WriteOptions(), &wb);
```