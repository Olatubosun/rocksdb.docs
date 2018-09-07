In this wiki we explain how files are deleted if they are not needed

# SST Files
When compaction finishes, the input SST files are replaced by the output ones in the LSM-tree. However, they may not qualify for being deleted immediately. Ongoing operations depending the older version of the LSM-tree will prevent those files being qualified to be dropped, until those operations finish. See [[How we keep track of live SST files]] for how the versioning of LSM-tree works.

The operations which can hold an older version of LSM-tree include:
1. Live iterators. Iterators pin the version of LSM-tree while they are created. All SST files from this version are prevented from being deleted. This is because an iterator reads data from a virtual snapshot, all SST files at the moment the iterator are preserved for data from the virtual snapshot to be available.
2. Ongoing compaction. Even if other compactions aren't compacting those SST files, the whole version of the LSM-tree is pinned.
3. Short window during Get(). In the short time a Get() is executed, the LSM-tree version is pinned to make sure it can read all the immutable SST files.
When no operation pins an old LSM-tree version containing an SST file anymore, the file is qualified to be deleted.

Those qualified files are deleted by two mechanism:

## Reference counting
RocksDB keeps a reference count for each SST file in memory. Each version of the LSM-tree holds one reference count for all the SST files in this version. The operations depending on this version (explained above) in turn hold reference count to the version of the LSM-tree directly or indirectly through "super version". Once the reference count of a version drops to 0, it drops the reference counts for all SST files in it. If a file's SST reference count drops to 0, it can be deleted. Usually they are deleted immediately, with two exceptions:
1. The file is found to be not needed when closing an iterator. If users set `ReadOptions.background_purge_on_iterator_cleanup=true`, rather than deleting the file immediately, we schedule a background job to the high-pri thread pool (the same pool where **flush jobs** are deleted) to delete the file.
2. 
Still continuing...