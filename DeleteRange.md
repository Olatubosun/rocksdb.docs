[This page is a WIP]

DeleteRange is an operation designed to replace the following pattern where a user wants to delete a range of keys in the range `[start, end)`:

```c++
...
Slice start, end;
// set start and end
auto it = db->NewIterator();

for (it->Seek(start); cmp->Compare(it->key(), end) < 0; it->Next()) {
  db->Delete(it->key());
}
...
```
 
This pattern requires performing a range scan, which prevents it from being usable on any performance-sensitive write path. To mitigate this, RocksDB provides a primitive operation to perform this task:
```c++
Slice start, end;
// set start and end
db->DeleteRange(start, end);
```

Under the hood, this creates a range tombstone represented as a single kv, which significantly speeds up write performance.

[TODO: talk about read performance]