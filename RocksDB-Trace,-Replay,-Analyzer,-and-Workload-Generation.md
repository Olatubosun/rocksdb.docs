# Untitled

# Query Tracing and Replaying

The trace_replay APIs allow the user to track down the query information to a trace file. In the current implementation, Get, WriteBatch (Put, Delete, Merge, SingleDelete, and DeleteRange), Iterator (Seek and SeekForPrev) are the queries that tracked by the trace_replay API. Key, query time stamp, value (if applied), cf_id forms one trace record. Since one lock is used to protect the tracing instance and there will be extra IOs for the trace file, the performance of DB will be influenced. According to the current test on the MyRocks and ZippyDB shadow server, the performance hasn't been a concern in those shadows. The trace records from the same DB instance are written to a binary trace file. User can specify the path of trace file (e.g., store in different storage devices to reduce the IO influence).

Currently, the trace file can be replayed by using the db_bench. The queries records in the traces file are replayed to the target DB instance according to the time stamps. It can replay the workload nearly the same as the workload being collected, which will provide a more production-like testing case. 

An simple example to use the tracing APIs:


```
Env* env = rocksdb::Env::Default();
EnvOptions env_options;
std::string trace_path = "/tmp/trace_test_example"
std::unique_ptr<TraceWriter> trace_writer;
DB* db = nullptr;
std::string db_name = "/tmp/rocksdb"

/*Create the trace file writer*/
NewFileTraceWriter(env, env_options, trace_path, &trace_writer);
DB::Open(options, dbname, &db);

/*Start tracing*/
db->StartTrace(trace_opt, std::move(trace_writer));

/* your call of RocksDB APIs */

/*End tracing*/
db->EndTrace()
```



To replay the trace:


```
./db_bench --benchmarks=replay --trace_file=/tmp/trace_test_example --num_column_families=5
```





# Trace Analyzing, Visualizing, and Modeling

After the user finishes the tracing steps by using the trace_replay APIs, the user will get one binary trace file. In the trace file, Get, Seek, and SeekForPrev are tracked with separate trace record, while queries of Put, Merge, Delete, SingleDelete, and DeleteRange are packed into WriteBatches. One tool is needed to 1) interpret the trace into the human readable format for further analyzing, 2) provide rich and powerful in-memory processing options to analyze the trace and output the corresponding results, and 3) be easy to add new analyzing options and query types to the tool.


The RocksDB team developed the initial version of the tool: trace_analyzer. It provide the following analyzing options and output results.

Note that most of the generated analyzing results output files will be separated in different column families and different query types, which means, one query type in one column family will have its own output files. Usually, one specified output option will generate one output file.


## Analyze The Trace

The trace analyer options

```
 -analyze_delete (Analyze the Delete query.) type: bool default: false
 -analyze_get (Analyze the Get query.) type: bool default: false
 -analyze_iterator ( Analyze the iterate query like seek() and
   seekForPrev().) type: bool default: false
 -analyze_merge (Analyze the Merge query.) type: bool default: false
 -analyze_put (Analyze the Put query.) type: bool default: false
 -analyze_range_delete (Analyze the DeleteRange query.) type: bool
   default: false
 -analyze_single_delete (Analyze the SingleDelete query.) type: bool
   default: false
 -convert_to_human_readable_trace (Convert the binary trace file to a human
   readable txt file for further processing. This file will be extremely
   large (similar size as the original binary trace file). You can specify
   'no_key' to reduce the size, if key is not needed in the next step
   File name: <prefix>_human_readable_trace.txt
   Format:[type_id cf_id value_size time_in_micorsec <key>].) type: bool
   default: false
 -key_space_dir (<the directory stores full key space files>
   The key space files should be: <column family id>.txt) type: string
   default: ""
 -no_key ( Does not output the key to the result files to make smaller.)
   type: bool default: false
 -no_print (Do not print out any result) type: bool default: false
 -output_access_count_stats (Output the access count distribution statistics
   to file.
   File name:  <prefix>-<query
   type>-<cf_id>-accessed_key_count_distribution.txt
   Format:[access_count number_of_access_count]) type: bool default: false
 -output_dir (The directory to store the output files.) type: string
   default: ""
 -output_ignore_count (<threshold>, ignores the access count <= this value,
   it will shorter the output.) type: int32 default: 0
 -output_key_distribution (Output the key size distribution.) type: bool
   default: false
 -output_key_stats (Output the key access count statistics to file
   for accessed keys:
   file name: <prefix>-<query type>-<cf_id>-accessed_key_stats.txt
   Format:[cf_id value_size access_keyid access_count]
   for the whole key space keys:
   File name: <prefix>-<query type>-<cf_id>-whole_key_stats.txt
   Format:[whole_key_space_keyid access_count]) type: bool default: false
 -output_prefix (The prefix used for all the output files.) type: string
   default: "trace"
 -output_prefix_cut (The number of bytes as prefix to cut the keys.
   if it is enabled, it will generate the following:
   for accessed keys:
   File name: <prefix>-<query type>-<cf_id>-accessed_key_prefix_cut.txt
   Format:[acessed_keyid access_count_of_prefix number_of_keys_in_prefix
   average_key_access prefix_succ_ratio prefix]
   for whole key space keys:
   File name: <prefix>-<query type>-<cf_id>-whole_key_prefix_cut.txt
   Format:[start_keyid_in_whole_keyspace prefix]
   if 'output_qps_stats' and 'top_k' are enabled, it will output:
   File name: <prefix>-<query
   type>-<cf_id>-accessed_top_k_qps_prefix_cut.txt
   Format:[the_top_ith_qps_time QPS], [prefix qps_of_this_second].)
   type: int32 default: 0
 -output_qps_stats (Output the query per second(qps) statistics
   For the overall qps, it will contain all qps of each query type. The time
   is started from the first trace record
   File name: <prefix>_qps_stats.txt
   Format: [qps_type_1 qps_type_2 ...... overall_qps]
   For each cf and query, it will have its own qps output
   File name: <prefix>-<query type>-<cf_id>_qps_stats.txt
   Format:[query_count_in_this_second].) type: bool default: false
 -output_time_series (Output the access time in second of each key, such
   that we can have the time series data of the queries
   File name: <prefix>-<query type>-<cf_id>-time_series.txt
   Format:[type_id time_in_sec access_keyid].) type: bool default: false
 -output_value_distribution (Out put the value size distribution, only
   available for Put and Merge.
   File name: <prefix>-<query
   type>-<cf_id>-accessed_value_size_distribution.txt
   Format:[Number_of_value_size_between x and x+value_interval is: <the
   count>]) type: bool default: false
 -print_correlation (intput format: [correlation pairs][.,.]
   Output the query correlations between the pairs of query types listed in
   the parameter, input should select the operations from:
   get, put, delete, single_delete, rangle_delete, merge. No space between
   the pairs separated by commar. Example: =[get,get]... It will print out
   the number of pairs of 'A after B' and the average time interval between
   the two query) type: string default: ""
 -print_overall_stats ( Print the stats of the whole trace, like total
   requests, keys, and etc.) type: bool default: true
 -print_top_k_access (<top K of the variables to be printed> Print the top k
   accessed keys, top k accessed prefix and etc.) type: int32 default: 1
 -trace_path (The trace file path.) type: string default: ""
 -value_interval (To output the value distribution, we need to set the value
   intervals and make the statistic of the value size distribution in
   different intervals. The default is 8.) type: int32 default: 8
```

**One Example**

```
./trace_analyzer -analyze_get -output_access_count_stats -output_dir=/data/trace/result -output_key_stats -output_qps_stats -convert_to_human_readable_trace -output_value_distribution -output_key_distribution -print_overall_stats -print_top_k_access=3 -output_prefix=test -trace_path=/data/trace/trace
```


**Query Type Options**

User can specify which type queries that should be analyzed and use “-analyze_<type>”.

**Output Human Readable Traces**

The original binary trace stores the encoded data structures and content, to interpret the trace, the tool should use the RocksDB library. Thus, to simplify the further analyzing of the trace, user can specify


```
-convert_to_human_readable_trace
```


The original trace will be converted to a txt file, the content is “[type_id cf_id value_size time_in_micorsec <key>]”. If the key is not needed, user can specify “-no_key” to reduce the file size. This option is independent to all other option, once it is specified, the converted trace will be generated. If the original key is included, the txt file size might be similar or even larger than the original trace file size.

**Input and Output Options**

To analyze a trace file, user need to indicate the path to the trace file by


```
-trace_path=<path to the trace>
```


To store the output files,  user can specify a directory (make sure the directory exist before running the analyzer) to store these files


```
-output_dir=<the path to the output directory>
```


If user wants to analyze the accessed keys together with the existing keyspace. User needs to specify a directory that stores the keyspace files. The file should be in the name of “<column family id>.txt” and each line is one key. Usually, user can use the “./ldb scan” of the LDB tool to dump out all the existing keys. To specify the directory


```
-key_space_dir=<the path to the key space directory>
```


To collect the output files more easily, user can specify the “prefix” for all the output files


```
-output_prefix=<the prefix, like "trace1">
```


If user does not want to print out the general statistics to the screen, user can specify 


```
-no_print
```



**The Analyzing Options**

Currently, the trace_analyzer tool provides several different analyzing options to characterize the workload. Some of the results are directly printed out (options with prefix “-print”) others will output to the files (options with prefix “-output”). User can specify the combination of the options to analyze the trace. Note that, some of the analyzing options are very memory intensive (e.g., -output_time_series, -print_correlation, and -key_space_dir). If the memory is not enough, try to run them in different time.

The general information of the workloads like the total number of keys, the number of analyzed queries of each column family, the key and value size statistics (average and medium), the number of keys being accessed are printed to screen when this option is specified


```
-print_overall_stats
```



To get the total access count of each key and the size of the value, user can specify 


```
-output_key_stats
```


It will output the access count of each key to a file and the keys are sorted in lexicographical order. Each key will be assigned with an integer as the ID for further processing

In some workloads, the composition of the keys has some common part. For example, in MyRocks, the first X bytes of the key is the table index_num. We can use the first X bytes to cut the keys into different prefix range. By specifying the number of bytes to cut the key space, the trace_analyzer will generate a file. One record in the file represents a cut of prefix, the corresponding KeyID, and the prefix content are stored. There will be two separate files if the -key_space_dir is specified. One file is for the accessed keys, the other one is for the whole key space. Usually, the prefix cut file is used together with the accessed_key_stats.txt and whole_key_stats.txt respectively. 


```
-output_prefix_cut=<number of bytes as prefix>
```



If user wants to visualize the accesses of the keys in the tracing timeline, user can specify:


```
-output_time_series
```


Each access to one key will be stored as one record in the time series file. 


If the user is interested to know about the detailed QPS changing during the tracing time, user can specify:


```
-output_qps_stats
```


For each query type of one column family, one file with the query number per second will be generated. Also, one file with the QPS of each query type on all column families as well as the overall QPS are output to a separate file. The average QPS and peak QPS will be printed out.

Sometimes, user might be interested to know about the TOP statistics. user can specify


```
-print_top_k_access
```


The top K accessed keys, the access number will be printed. Also, if the prefix_cut option is specified, the top K accessed prefix with their total access count are printed. At the same time, the top K prefix with the highest average access is printed. 

If the user is interested to know about the value size distribution (only applicable for Put and Merge ) user can specify


```
-output_value_distribution
-value_interval
```


Since the value size varies a lot, User might just want to know how many values are in each value size range. User can specify the value_interval=x to generate the number of values between [0,x), [x,2x)......

The key size distribution is output to the file if user specify


```
-output_key_distribution
```



## Visualize the workload

After processing the trace with trace_analyzer, user will get a couple of output files. Some of the files can be used to visualize the workload such as the heatmap (the hotness or coldness of keys), time series graph (the overview of the key access in timeline), the QPS of the analyzed queries. 

Here, we use the open source plot tool GNUPLOT as an example to generate the graphs. More details about the GNUPLOT can be found here (http://gnuplot.info/). User can directly write the GNUPLOT command to draw the graph, or to make it simple, user can use the following shell script to generate the GNUPLOT source file (before using the script, make sure the file name and some of the content are replaced with the effective ones).

To draw the heat map of accessed keys


```
#!/bin/bash

# The query type
ops="iterator"

# The column family ID
cf="9"

# The column family name if known, if not, replace it with some prefix
cf_name="rev:cf-assoc-deleter-id1-type"

form="accessed"

# The column number that will be plotted
use="4"

# The higher bound of Y-axis
y=2

# The higher bound of X-axis
x=29233
echo "set output '${cf_name}-${ops}-${form}-key_heatmap.png'" > plot-${cf_name}-${ops}-${form}-heatmap.gp
echo "set term png size 2000,500" >>plot-${cf_name}-${ops}-${form}-heatmap.gp
echo "set title 'CF: ${cf_name} ${form} Key Space Heat Map'">>plot-${cf_name}-${ops}-${form}-heatmap.gp
echo "set xlabel 'Key Sequence'">>plot-${cf_name}-${ops}-${form}-heatmap.gp
echo "set ylabel 'Key access count'">>plot-${cf_name}-${ops}-${form}-heatmap.gp
echo "set yrange [0:$y]">>plot-${cf_name}-${ops}-${form}-heatmap.gp
echo "set xrange [0:$x]">>plot-${cf_name}-${ops}-${form}-heatmap.gp

# If the preifx cut is avialable, it will draw the prefix cut
while read f1 f2
do
    echo "set arrow from $f1,0 to $f1,$y nohead lc rgb 'red'" >> plot-${cf_name}-${ops}-${form}-heatmap.gp
done < "trace.1532381594728669-${ops}-${cf}-${form}_key_prefix_cut.txt"
echo "plot 'trace.1532381594728669-${ops}-${cf}-${form}_key_stats.txt' using ${use} notitle w dots lt 2" >>plot-${cf_name}-${ops}-${form}-heatmap.gp
gnuplot plot-${cf_name}-${ops}-${form}-heatmap.gp
```



To draw the time series map of accessed keys


```
#!/bin/bash

# The query type
ops="iterator"

# The higher bound of X-axis
x=29233

# The column family ID
cf="8"

# The column family name if known, if not, replace it with some prefix
cf_name="rev:cf-assoc-deleter-id1-type"

# The type of the output file
form="time_series"

# The column number that will be plotted
use="3:2"

# The total time of the tracing duration, in seconds
y=88000
echo "set output '${cf_name}-${ops}-${form}-key_heatmap.png'" > plot-${cf_name}-${ops}-${form}-heatmap.gp
echo "set term png size 3000,3000" >>plot-${cf_name}-${ops}-${form}-heatmap.gp
echo "set title 'CF: ${cf_name} time series'">>plot-${cf_name}-${ops}-${form}-heatmap.gp
echo "set xlabel 'Key Sequence'">>plot-${cf_name}-${ops}-${form}-heatmap.gp
echo "set ylabel 'Key access count'">>plot-${cf_name}-${ops}-${form}-heatmap.gp
echo "set yrange [0:$y]">>plot-${cf_name}-${ops}-${form}-heatmap.gp
echo "set xrange [0:$x]">>plot-${cf_name}-${ops}-${form}-heatmap.gp

# If the preifx cut is avialable, it will draw the prefix cut
while read f1 f2
do
    echo "set arrow from $f1,0 to $f1,$y nohead lc rgb 'red'" >> plot-${cf_name}-${ops}-${form}-heatmap.gp
done < "trace.1532381594728669-${ops}-${cf}-accessed_key_prefix_cut.txt"
echo "plot 'trace.1532381594728669-${ops}-${cf}-${form}.txt' using ${use} notitle w dots lt 2" >>plot-${cf_name}-${ops}-${form}-heatmap.gp
gnuplot plot-${cf_name}-${ops}-${form}-heatmap.gp
```



To plot out the QPS


```
#!/bin/bash

# The query type
ops="iterator"

# The higher bound of the QPS
y=5

# The column family ID
cf="9"

# The column family name if known, if not, replace it with some prefix
cf_name="rev:cf-assoc-deleter-id1-type"

# The type of the output file
form="qps_stats"

# The column number that will be plotted
use="1"

# The total time of the tracing duration, in seconds
x=88000
echo "set output '${cf_name}-${ops}-${form}-IO_per_second.png'" > plot-${cf_name}-${ops}-${form}-heatmap.gp
echo "set term png size 2000,1200" >>plot-${cf_name}-${ops}-${form}-heatmap.gp
echo "set title 'CF: ${cf_name} QPS Over Time'">>plot-${cf_name}-${ops}-${form}-heatmap.gp
echo "set xlabel 'Time in second'">>plot-${cf_name}-${ops}-${form}-heatmap.gp
echo "set ylabel 'QPS'">>plot-${cf_name}-${ops}-${form}-heatmap.gp
echo "set yrange [0:$y]">>plot-${cf_name}-${ops}-${form}-heatmap.gp
echo "set xrange [0:$x]">>plot-${cf_name}-${ops}-${form}-heatmap.gp
echo "plot 'trace.1532381594728669-${ops}-${cf}-${form}.txt' using ${use} notitle with linespoints" >>plot-${cf_name}-${ops}-${form}-heatmap.gp
gnuplot plot-${cf_name}-${ops}-${form}-heatmap.gp
```





## Model the Workloads

We can use different tools, script, and models to fit the workload statistics. Typically, user can use the distributions of key access count and prefix access count to fit in the model. Also, QPS can be modeled. Here, we use Matlab as an example to fit the key access count, prefix access count, and QPS.

User can try different distributions to fit the model. In this example, we use two-term exponential model to fit the access distribution, and two-term sin() to fit the QPS

To fit the key access statistics to the model and get the statistics, user can run the following script:


```
% This script is used to fit the key access count distribution
% to the two-term exponential distirbution and get the parameters

% The input file with surfix: accessed_key_stats.txt
fileID = fopen('trace.1531329742187378-get-4-accessed_key_stats.txt');
txt = textscan(fileID,'%f %f %f %f');
fclose(fileID);

% Get the number of keys that has access count x
t2=sort(txt{4},'descend');

% The number of access count that is used to fit the data
% The value depends on the accuracy demond of your model fitting
% and the value of count should be always not greater than
% the size of t2
count=30000;

% Generate the access count x
x=1:1:count;
x=x';

% Adjust the matrix and uniformed
y=t2(1:count);
y=y/(sum(y));
figure;

% fitting the data to the exp2 model
f=fit(x,y,'exp2')

%plot out the original data and fitted line to compare
plot(f,x,y);
```


To fit the key access count distribution to the model, user can run the following script:


```
% This script is used to fit the key access count distribution
% to the two-term exponential distirbution and get the parameters

% The input file with surfix: key_count_distribution.txt
fileID = fopen('trace-get-9-accessed_key_count_distribution.txt');
input = textscan(fileID,'%s %f %s %f');
fclose(fileID);

% Get the number of keys that has access count x
t2=sort(input{4},'descend');

% The number of access count that is used to fit the data
% The value depends on the accuracy demond of your model fitting
% and the value of count should be always not greater than
% the size of t2
count=100;

% Generate the access count x
x=1:1:count;
x=x';
y=t2(1:count);

% Adjust the matrix and uniformed
y=y/(sum(y));
y=y(1:count);
x=x(1:count);

figure;
% fitting the data to the exp2 model
f=fit(x,y,'exp2')

%plot out the original data and fitted line to compare
plot(f,x,y);
```



To fit the prefix average access count to the model, user can run the following script:


```
% This script is used to fit the prefix average access count distribution
% to the two-term exponential distirbution and get the parameters

% The input file with surfix: accessed_key_prefix_cut.txt
fileID = fopen('trace-get-4-accessed_key_prefix_cut.txt');
txt = textscan(fileID,'%f %f %f %f %s');
fclose(fileID);

% The per key access (average) of each prefix, sorted
t2=sort(txt{4},'descend');

% The number of access count that is used to fit the data
% The value depends on the accuracy demond of your model fitting
% and the value of count should be always not greater than
% the size of t2
count=1000;

% Generate the access count x
x=1:1:count;
x=x';

% Adjust the matrix and uniformed
y=t2(0:count);
y=y/(sum(y));
x=x(1:count);

% fitting the data to the exp2 model
figure;
f=fit(x,y,'exp2')

%plot out the original data and fitted line to compare
plot(f,x,y);
```



To fit the QPS to the model, user can run the following script:


```
% This script is used to fit the qps of the one query in one of the column
% family to the sin'x' model. 'x' can be 1 to 10. With the higher value
% of the 'x', you can get more accurate fitting of the qps. However,
% the model will be more complex and some times will be overfitted.
% The suggestion is to use sin1 or sin2

% The input file shoud with surfix: qps_stats.txt
fileID = fopen('trace-get-4-io_stats.txt');
txt = textscan(fileID,'%f');
fclose(fileID);
t1=txt{1};

% The input is the queries per second. If you directly use the qps
% you may got a high value of noise. Here, 'n' is the number of qps
% that you want to combined to one average value, such that you can
% reduce it to queries per n*seconds.
n=10;
s1 = size(t1, 1);
M  = s1 - mod(s1, n);
t2  = reshape(t1(1:M), n, []);
y = transpose(sum(t2, 1) / n);

% Up to this point, you need to move the data down to the x-axis,
% the offset is the ave. So the model will be 
% s(x) =  a1*sin(b1*x+c1) + a2*sin(b2*x+c2) + ave
ave = mean(y);
y=y-ave;

% Adjust the matrix
count = size(y,1);
x=1:1:count;
x=x';

% Fit the model to 'sin2' in this example and draw the point and
% fitted line to compare
figure;
s = fit(x,y,'sin2')
plot(s,x,y);
```


Users can use the model for further analyzing or use it to generate the synthetic workload.

# Synthetic Workload Generation based on Models

In the previous section, users can use the fitting functions of Matlab to fit the traced workload to different models, such that we can use a set of parameters and functions to profile the workload. We focus on the 4 variables to profile the workload: 1) value size; 2) KV-pair Access; 3) QPS; 4) Iterator scan length. According to our current research, the value size and Iterator scan length follows the Generalized Pareto Distribution. The probability density function is: f(x) = (1/sigma)*(1+k*(x-theta)\sigma)^(-1-1/k). The KV-pair access follows power-law, in which about 80% of the KV-pair has an access count less than 4. We sort the keys based on the access count in descending order and fit them to the models. The two-term power model fit the KV-pair access distribution best. The probability density function is: f(x) = ax^b+c. The Sine function fits the QPS best. F(x) = A*sin(B*x + C) + D.

Here is one example of the parameters we get from the workload collected in Facebook social graph:

1) Value Size: sigma = 226.409, k = 0.923$, theta = 0
2) KV-pair Access: a = 0.001636, b = -0.7094 , and c = 3.217*10^-9
3) QPS: $A = 147.9, B = 8.3*10^-5, C = -1.734, D = 1064.2
4) Iterator scan length: sigma = 1.747, k = 0.0819, theta = 0

We developed an benchmark called “mixgraph” in db_bench, which can use the four set of parameters to generate the synthetic workload. The workload is statistically similar to the original one. Note that, only the workload that can be fit to the models used for the four variables can be used in the mixgraph. For example, if the value size follows the power distribution instead of Generalized Pareto Distribution, we cannot use the mixgraph to generate the workloads.

To enable the “mixgraph” benchmark, user needs to specify:

```
./db_bench —benchmarks="mixgraph"
```


To set the parameters of the value size distribution (Generalized Pareto Distribution only), user needs to specify:

```
-value_k=<> -value_sigma=<> -value_theta=<>
```


To set the parameters of the KV-pair access distribution (power distribution only and C==0), user needs to specify:

```
-key_dist_a=<> -key_dist_b=<>
```


To set the parameters of the QPS (Sine),  user needs to specify:

```
-sine_a=<> -sine_b=<> -sine_c=<> -sine_d=<> -sine_mix_rate_interval_milliseconds=<>
```

The mix rate is used to set the time interval that how lone we should correct the rate according to the distribution, the smaller it is, the better it will fit.

To set the parameters of the iterator scan length distribution (Generalized Pareto Distribution only), user needs to specify:

```
-iter_k=<> -iter_sigma=<> -iter_theta=<>
```


User need to specify the query ratio between the Get, Put, and Seek. Such that we can generate the mixed workload that can be similar to the social graph workload (so called mix graph)

```
-mix_get_ratio=<> -mix_put_ratio=<> -mix_seek_ratio=<>
```


Finally, user need to specify how many queries they want to execute:

```
-reads=<>
```

and what's the total KV-pairs are in the current testing DB

```
-num=<>
```

The num together with the aforementioned distributions decided the queries.

Here is one example that can be directly used to generate the workload in the DB with 5000000 KV-pairs with 1000000 queries:

```
./db_bench --benchmarks="mixgraph"  -value_k=0.1033 -value_sigma=39 -key_dist_a=0.002312 -key_dist_b=0.3467 -sine_mix_rate_interval_milliseconds=500 -sine_a=350 -sine_b=0.0105 -sine_d=2300 -iter_k=2.517 -iter_sigma=14.236 -mix_get_ratio=0.806 -mix_put_ratio=0.159 -mix_seek_ratio=0.035 -reads=1000000 -num=5000000
```







