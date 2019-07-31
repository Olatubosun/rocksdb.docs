RocksDB configures a certain amount of main memory as a block cache to accelerate data access. Understanding the efficiency of block cache is very important. The block cache analysis and simulation tools help a user to collect block cache access traces, analyze its access pattern, and evaluate alternative caching policies. 

### Table of Contents
* **[Quick Start](#quick-start)**<br>
* **[Tracing block cache accesses](#tracing-block-cache-accesses)**<br>
* **[Trace Format](#trace-format)**<br>
* **[Cache Simulations](#cache-simulations)**<br>
  * [RocksDB Cache Simulators](#rocksdb-cache-simulators)<br>
  * [Python Cache Simulators](#python-cache-simulators)<br>
    * [Supported Cache Simulators](#supported-cache-simulators)<br>
* **[Analyzing Block Cache Traces](#analyzing-block-cache-traces)**<br>

# Quick Start
db_bench supports tracing block cache accesses. This section demonstrates how to trace accesses when running db_bench. It also shows how to analyze and evaluate caching policies using the generated trace file. 

Create a database: 
```
./db_bench --benchmarks="fillseq" --key_size=20 --prefix_size=20 --keys_per_prefix=0 --value_size=100 --cache_index_and_filter_blocks --cache_size=1048576 --disable_auto_compactions=1 --disable_wal=1 --compression_type=none --min_level_to_compress=-1 --compression_ratio=1 --num=10000000
```
To trace block cache accesses when running `readrandom` benchmark:
```
./db_bench --benchmarks="readrandom" --use_existing_db --duration=60 --key_size=20 --prefix_size=20 --keys_per_prefix=0 --value_size=100 --cache_index_and_filter_blocks --cache_size=1048576 --disable_auto_compactions=1 --disable_wal=1 --compression_type=none --min_level_to_compress=-1 --compression_ratio=1 --num=10000000 --threads=16 -block_cache_trace_file="/tmp/binary_trace_test_example" -block_cache_trace_max_trace_file_size_in_bytes=1073741824 -block_cache_trace_sampling_frequency=1
```
Convert the trace file to human readable format:
```
./block_cache_trace_analyzer -block_cache_trace_path=/tmp/binary_trace_test_example -human_readable_trace_file_path=/tmp/human_readable_block_trace_test_example
```
Evaluate alternative caching policies: 
```
bash block_cache_pysim.sh /tmp/human_readable_block_trace_test_example /tmp/sim_results/bench 1 0 30
```
Plot graphs:
```
python block_cache_trace_analyzer_plot.py /tmp/sim_results /tmp/sim_results_graphs
```

# Tracing Block Cache Accesses
RocksDB supports block cache tracing APIs `StartBlockCacheTrace` and `EndBlockCacheTrace`. When tracing starts, RocksDB logs detailed information of block cache accesses into a trace file. A user must specify a trace option and trace file path when start tracing block cache accesses.

A trace option contains `max_trace_file_size` and `sampling_frequency`.
- `max_trace_file_size` specifies the maximum size of the trace file. The tracing stops when the trace file size exceeds the specified `max_trace_file_size`.
- `sampling_frequency` determines how frequent should RocksDB trace an access. RocksDB uses spatial downsampling such that it traces all accesses to sampled blocks. A `sampling_frequency` of 1 means tracing all block cache accesses. A `sampling_frequency` of 100 means tracing all accesses on ~1% blocks. 

An example to start tracing block cache accesses: 
```
Env* env = rocksdb::Env::Default();
EnvOptions env_options;
std::string trace_path = "/tmp/binary_trace_test_example"
std::unique_ptr<TraceWriter> trace_writer;
DB* db = nullptr;
std::string db_name = "/tmp/rocksdb"

/*Create the trace file writer*/
NewFileTraceWriter(env, env_options, trace_path, &trace_writer);
DB::Open(options, dbname, &db);

/*Start tracing*/
db->StartBlockCacheTrace(trace_opt, std::move(trace_writer));

/* Your call of RocksDB APIs */

/*End tracing*/
db->EndBlockCacheTrace()
```

# Trace Format
We can convert the generated binary trace file into human readable trace file in csv format. It contains the following columns. 

| Column Name   |  Values     | Comment |
| :------------- |:-------------|:-------------|
| Access timestamp in microseconds     | unsigned long | |
| Block ID      |  unsigned long     | A unique block ID |
| Block type | 7: Index block <br> 8: Filter block <br> 9: Data block <br> 10: Uncompressed dictionary block <br> 11: Range deletion block   | |
| Block size | unsigned long     | |
| Column family ID | unsigned long | A unique column family ID | 
| Column family name | string   | |
| Level | unsigned long  | The LSM tree level of this block |
| SST file number | unsigned long     | The SST file this block belongs to | 
| Caller |  See [Caller](https://github.com/facebook/rocksdb/blob/master/table/table_reader_caller.h) | The caller that accesses this block |
| No insert | 0: do not insert the block upon a miss <br> 1: insert the block upon a cache miss   |  | 
| Get ID | unsigned long | A unique ID associated with a Get request |
| Get key ID | unsigned long | The referenced key of the Get request |
| Get referenced data size | unsigned long | The referenced data (key+value) size of the Get request |
| Is a cache hit |  0: A cache hit <br> 1: A cache miss    | The running RocksDB instance observes a cache hit/miss on this block |
| Does get referenced key exist in this block | 0: Does not exist <br> 1: Exist      | Data block only: Whether the referenced key is found in this block. |
| Approximate number of keys in this block | unsigned long | Data block only |
| Get table ID | unsigned long     | The table ID of the get request. We treat the first four bytes of the Get request as table ID |
| Get sequence number | unsigned long | The sequence number associated with the Get request |
| Block key size | unsigned long     |  |
| Get referenced key size | unsigned long     | |
| Block offset in the SST file | unsigned long  | |

# Cache Simulations
We support running cache simulators using both RocksDB built-in caches and caching policies written in python. 
## RocksDB Cache Simulators
To replay the trace and evaluate alternative policies, we first need to provide a cache configuration file. An example file contains the following content: 
```
lru,0,0,16M,256M,1G,2G,4G,8G,12G,16G,1T
```
| Column Name   |  Values     |
| :------------- |:-------------|
| Cache name     | lru: LRU <br> lru_priority: LRU with midpoint insertion <br> lru_hybrid: LRU that also caches row keys <br> ghost_*: A ghost cache for admission control  |
| Number of shard bits      |  unsigned long     |
| Ghost cache capacity      |  unsigned long     |
| Cache sizes      |  A list of comma separated cache sizes      |

Next, we can start simulating caches. 
```
./block_cache_trace_analyzer -mrc_only=true -block_cache_trace_downsample_ratio=100 -block_cache_trace_path=/tmp/binary_trace_test_example -block_cache_sim_config_path=/tmp/cache_config -block_cache_analysis_result_dir=/tmp/binary_trace_test_example_results -cache_sim_warmup_seconds=3600
```

It contains two important parameters: 

`block_cache_trace_downsample_ratio`: The sampling frequency we used to collect trace. The simulator will scale down the given cache size by this factor. For example, with downsample_ratio of 100, the cache simulator will create a 1 GB cache to simulate a 100 GB cache. 

`cache_sim_warmup_seconds`: The number of seconds used for warmup. 

The analyzer outputs a few files: 
```
TODO.
```

## Python Cache Simulators
We need to first convert the binary trace file into human readable trace file. 
```
./block_cache_trace_analyzer -block_cache_trace_path=/tmp/binary_trace_test_example -human_readable_trace_file_path=/tmp/human_readable_block_trace_test_example
```

Then, we can simulate a cache as follows: 
```
python block_cache_pysim.py lru 16M 100 3600 /tmp/human_readable_block_trace_test_example /tmp/results 10000000 0 all 
```
To simulate a batch of cache configurations: 
```
bash block_cache_pysim.sh /tmp/human_readable_block_trace_test_example /tmp/sim_results/bench 1 0 30
```
A block_cache_pysim.py output the following files: 

A block_cache_pysim.sh combines outputs of block_cache_pysim.py into following files: 

### Supported Cache Simulators 

| Cache Name   |  Comment     |
| :------------- |:-------------|
| lru | Strict (Least recently used) LRU cache. The cache maintains an LRU queue. |
| gdsize | GreedyDual Size. <br> N. Young. The k-server dual and loose competitiveness for paging. Algorithmica, June 1994, vol. 11,(no.6):525-41. Rewritten version of ''On-line caching as cache size varies'', in The 2nd Annual ACM-SIAM Symposium on Discrete Algorithms, 241-250, 1991. |
| opt | The Belady MIN algorithm.<br> L. A. Belady. 1966. A Study of Replacement Algorithms for a Virtual-storage Computer. IBM Syst. J. 5, 2 (June 1966), 78-101. DOI=http://dx.doi.org/10.1147/sj.52.0078 |
| arc | Adaptive replacement cache.<br> Nimrod Megiddo and Dharmendra S. Modha. 2003. ARC: A Self-Tuning, Low Overhead Replacement Cache. In Proceedings of the 2nd USENIX Conference on File and Storage Technologies (FAST '03). USENIX Association, Berkeley, CA, USA, 115-130. |
| pylru | LRU cache with random sampling. |
| pymru | (Most recently used) MRU cache with random sampling. |
| pylfu | (Least frequently used) LFU cache with random sampling. |
| pyhb | Hyperbolic Caching.<br> Aaron Blankstein, Siddhartha Sen, and Michael J. Freedman. 2017. Hyperbolic caching: flexible caching for web applications. In Proceedings of the 2017 USENIX Conference on Usenix Annual Technical Conference (USENIX ATC '17). USENIX Association, Berkeley, CA, USA, 499-511. |
| pyccbt | Cost class: block type  |
| pycccfbt | Cost class: column family + block type |
| ts | Thompson sampling <br> Daniel J. Russo, Benjamin Van Roy, Abbas Kazerouni, Ian Osband, and Zheng Wen. 2018. A Tutorial on Thompson Sampling. Found. Trends Mach. Learn. 11, 1 (July 2018), 1-96. DOI: https://doi.org/10.1561/2200000070 |
| linucb | Linear UCB <br> Lihong Li, Wei Chu, John Langford, and Robert E. Schapire. 2010. A contextual-bandit approach to personalized news article recommendation. In Proceedings of the 19th international conference on World wide web (WWW '10). ACM, New York, NY, USA, 661-670. DOI=http://dx.doi.org/10.1145/1772690.1772758 |
| trace | Trace |
| *_hybrid | A hybrid cache that also caches row keys. |

`py*` caches uses random sampling at eviction time. It samples 64 random entries in the cache, sorts these entries based on a priority function, e.g., LRU, and evicts from the lowest priority entry until the cache has enough capacity to insert the new entry.

`pycc*` caches group cached entries by a cost class. The cache maintains aggregated statistics for each cost class such as number of hits, total size. A cached entry is also tagged with one cost class. At eviction time, the cache samples 64 random entries and group them by their cost class. It then evicts entries based on their cost class's statistics. 

`ts` and `linucb` are two caches using reinforcement learning. The cache is configured with N policies, e.g., LRU, MRU, LFU, etc. The cache learns which policy is the best overtime and selects the best policy for eviction. The cache rewards the selected policy if it has not evicted the new key before. 
`ts` does not use any feature of a block while `linucb` uses three features: a block's level, column family, and block type. 

`trace` reports the misses observed in the collected trace. 

# Analyzing Block Cache Traces
Provides insights into how to improve a caching policy. 




