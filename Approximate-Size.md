The approximate size APIs allow a user to get a reasonably accurate guess of disk space and memory utilization of a key range.

# API Usage

The main APIs are ```GetApproximateSizes()``` and ```GetApproximateMemTableStats()```. The former takes a ```struct SizeApproximationOptions``` as an argument. It has the following fields -
* ```include_memtabtles``` - Indicates whether to count the memory usage of a given key range in memtables towards the overall size of the key range.
* ```include_files``` - Indicates whether to count the size of SST files occupied by the key range towards the overall size. At least one of this or ```include_memtabtles``` must be set to ```true```.
* ```files_size_error_margin``` - This option indicates the acceptable ratio of over/under estimation of file size to the actual file size. For example, a value of ```0.1``` means the approximate size will be within 10% of the actual size of the key range. The main purpose of this option is to make the calculation more efficient. Setting this to ```-1.0``` will force RocksDB to seek into SST files to accurately calculate the size, which will be more CPU intensive.

Example,
```cpp
  std::array<Range, NUM_RANGES> ranges;
  std::array<uint64_t, NUM_RANGES> sizes;
  SizeApproximationOptions options;

  options.include_memtabtles = true;
  options.files_size_error_margin = 0.1;

  ranges[0].start = start_key1;
  ranges[0].limit = end_key1;
  ranges[1].start = start_key2;
  ranges[1].limit = end_key2;

  Status s = GetApproximateSizes(options, column_family, ranges.data(), NUM_RANGES, sizes.data());
  // sizes[0] and sizes[1] contain the size in bytes for the respective ranges
```

The API counterpart for memtable usage is ```GetApproximateMemTableStats```, which returns the number of entries and total size of a given key range. Example,

```cpp
  Range range;
  uint64_t count;
  uint64_t size;

  range.start = start_key;
  range.limit = end_key;

  Status s = GetApproximateMemTableStats(column_family, range, &count, &size);
```

The ```GetApproximateMemTableStats``` is only supported for memtables created by ```SkipListFactory```.

Note that the approximate size from SST files are size of compressed blocks. It might be significantly smaller than the actual key/value size.