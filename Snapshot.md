A snapshot captures a point-in-time view of the DB at the time it's created. Snapshots do not persist across DB restarts.

## API Usage

- Create a snapshot with the `GetSnapshot()` API.
- Read from a snapshot by setting `ReadOptions::snapshot`.
- When finished, release resources associated with the snapshot by calling `ReleaseSnapshot()`.

## Implementation

### Flush/compaction

### Representation

### Scalability