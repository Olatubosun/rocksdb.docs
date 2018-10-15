A snapshot captures a point-in-time view of the DB at the time it's created. Snapshots do not persist across DB restarts.

## API Usage

- Create a snapshot with the `GetSnapshot()` API.
- Read from a snapshot by setting `ReadOptions::snapshot`.
- When finished, release resources associated with the snapshot by calling `ReleaseSnapshot()`.

## Implementation

### Flush/compaction

### Representation

A snapshot is represented by a small object of `SnapshotImpl` class. It holds only a few primitive fields, like the seqnum at which the snapshot was taken.

Snapshots are stored in a linked list owned by `DBImpl`. One benefit is we can allocate the list node before acquiring the DB mutex. Then while holding the mutex, we only need to update list pointers. Additionally, `ReleaseSnapshot()` can be called on the snapshots in an arbitrary order. With linked list, we can remove a node from the middle without shifting.

### Scalability

The main downside of linked list is it cannot be binary searched despite its ordering. During flush/compaction, we have to scan the snapshot list when we need to find out the earliest snapshot to which a key is visible. When there are many snapshots, this scan can significantly slow down flush/compaction to the point of causing write stalls. We've noticed problems when snapshot count is in the hundreds of thousands.