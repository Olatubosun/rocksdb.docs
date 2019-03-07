## Purpose

Dynamic dictionary compression algorithms have trouble compressing small data. With default usage, the compression dictionary starts off empty and is constructed during a single pass over the input data. Thus small input leads to a small dictionary, which cannot achieve good compression ratios.

In RocksDB, each block in a block-based table (SST file) is compressed individually. Block size defaults to 4KB, from which the compression algorithm cannot build a sizable dictionary. The dictionary compression feature presets the dictionary with data sampled from multiple blocks, which improves compression ratio when there are repetitions across blocks.

## Usage

Set `rocksdb::CompressionOptions::max_dict_bytes` (in `include/rocksdb/options.h`) to a nonzero value indicating the maximum per-file dictionary size.

Also `rocksdb::CompressionOptions::zstd_max_train_bytes` may be used to generate a training dictionary of max bytes for ZStd compression. Using ZStd's dictionary trainer can achieve even better compression ratio improvements than using `max_dict_bytes` alone. The training data will be used to generate a dictionary of `max_dict_bytes`.

## Implementation

Dictionary compression is only implemented for the bottom-most level. The dictionary will be constructed for each SST file in a subcompaction by sampling entire data blocks in the file. When dictionary compression is enabled, the uncompressed data blocks in the file being generated will be buffered in memory, upto ```target_file_size``` bytes. Once the limit is reached, or the file is finished, data blocks are taken uniformly/randomly from the buffered data blocks and used to train the ZStd dictionary trainer.

The dictionary is stored in the file's meta-block in order for it to be known when uncompressing. During reads, if ```BlockBasedTableOptions::cache_index_and_filter_blocks``` is ```true```, the dictionary meta-block is cached in the block cache, though it is not prefetched unlike index and filter blocks. If it is ```false```, the dictionary meta-block is read and pinned in memory , but not charged to the block cache.

The in-memory uncompression dictionary is a digested form of the raw dictionary stored on disk, and is larger in size. The digested form makes uncompression faster, but does consume more memory.

## Limitations

* Applies only to bottommost level
* Dictionary meta-block is not prefetched into the block cache
