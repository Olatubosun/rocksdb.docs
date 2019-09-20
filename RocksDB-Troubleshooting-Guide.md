## Information Logs
Usually, looking at information logs to see whether there are something abnormal is the first step of troubleshooting.

By default, RocksDB’s information logs residence in the same directory as your database. Look for `LOG` for the most recent log file, and `LOG.*` files for older ones. If you’ve explicitly set `option.db_log_dir`, find the logs there. The log file names will be contain information of the paths of your database.

Some users have overridden the default logger or use non-default ones and they’ll need tp find the logs accordingly.
If options.
Be aware that the default log level is `INFO`. With `DEBUG` level you’ll see more information, and with `WARN` level less.

With the default logger, a log line might look like this:
```
2019/09/17-13:42:20.597910 7fef48069300 [_impl/db_impl_compaction_flush.cc:1473] [default] Manual compaction starting
```
After the timestamp, `7fef48069300` is the thread ID. Since usually a flush or compaction happens in one same thread, the thread ID might help you correlate which lines belong to the same job. `_impl/db_impl_compaction_flush.cc:1473` shows the source file and line number where the line is logged. It might be cut short to avoid long log lines. `default` the column family name.

## Examine data on disk
If you see that RocksDB has returned unexpected results. You may consider to examine the database files and see whether the data on disk is wrong and if it is wrong in what way. `Ldb` is the best tool to do that (ldb stands for LevelDB and we never renamed it).

Start with examining the data with `get` and `scan`. These commands will only examine keys visible to users. If you want to examine invisible keys, e.g. tombstones, older versions, you can use `idump`. Another useful command is manifest_dump, which shows the LSM-tree structure of the database. This can help narrow down the problem to LSM-tree metadata or certain SST files.

If you use a customized comparator, merge operator or Env, you may need to build a customized `ldb` for it to work with your database. You can do it by building a binary which calls `LDBTool::Run()` with customized options, or register your plug in your plug in using object registry before calling the function. See `ldb_tools.h` for details. 

To examine a specific SST file, use `sst_dump` tool. Starting with scan to examine the logical data. If needed, command raw can help examine data in more details.

See --help and [[Administration and Data Access Tool]] for more details about the two tools.

## Debugging Performance Problems
It’s a good idea to start with statistics. [[Statistics]] and the information returned by `DB::GetProperty()` are two good places to look.Information Logs contain some performance information about each flush or compaction. For slow reads, [[Perf Context and IO Stats Context]] can help break down the latency. [[RocksDB Tuning Guide]] has some information about RocksDB performance.

## Report Issues
We use github issues only for bug reports, and use Google Group or Facebook Group for other issues. It’s not always clear to users whether it is RocksDB bug or not, pick one using your best judgement.

To help the community to help more efficiently, provide as much information as possible. Try to include the RocksDB release number, options used (e.g. paste the option file under the DB directory), what’s seen when examining the database with `ldb` or `sst_dump`, related statistics, etc. You can even consider to attach the information files. Data itself is not logged there.

For performance related questions, it may be helpful to check [[How to ask a performance related question]].
