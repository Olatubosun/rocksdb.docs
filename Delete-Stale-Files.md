In this wiki we explain how files are deleted if they are not needed

# SST Files
When compaction finishes, the input SST files are replaced by the output ones in the LSM-tree. However, they may not qualify for being deleted immediately. Ongoing operations depending the older version of the LSM-tree will prevent those files being qualified to be dropped, until those operations finish. The operations include:
1. Live iterators. Iterators pin the version of LSM-tree while they are created. All SST files from this version are prevented from being deleted. This is because an iterator reads data from a virtual snapshot, all SST files at the moment the iterator are preserved for data from the virtual snapshot to be available.
2. Ongoing compaction. Even if other compactions aren't compacting those SST files, the whole version of the LSM-tree is pinned.
3. Short window during Get(). In the short time a Get() is executed, the LSM-tree version is pinned to make sure it can read all the immutable SST files.
When no operation pins an old LSM-tree version containing an SST file anymore, the file is qualified to be deleted.