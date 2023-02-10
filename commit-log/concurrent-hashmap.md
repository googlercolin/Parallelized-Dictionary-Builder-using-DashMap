# [PR] Multithreaded `dictionary_builder` using concurrent HashMaps (`DashMap`)

# Summary
To improve the performance of the original sequential `dictionary_builder`, multithreading was implemented. This was done by splitting the original log file into chunks, and sending each chunk to be processed by separate threads. A concurrent HashMap, `DashMap`, was used across all threads. Code modifications were mainly done in `parser.rs`. However, because we want to make this version be the default version run, we also modified `main.rs` to read the `single_map` args.

# Tech details
We implement multithreading using the `ThreadPool v1.8.1` crate, and the concurrent Hashmap using the `DashMap v5.4.0` crate. We first create a specifiable number of chunks, and threads, depending on the `--num_threads` flag (default: 8). This simply splits the original log file into `num_threads` number of chunks.
Then using a `mpsc` channel, we send a `worker_conc` function which processes each chunk of logs, to the threads in the ThreadPool. As they are processed in their separate threads two `DashMap`s are used to update the key, value pairs for the `dbl` and `trpl` HashMaps, and a `DashSet` is used to concurrently update the `all_token_list` Vector.

The `worker_conc` function is almost identical to the old dictionary_builder function. The main difference is the return types. To perform the joining from the separate threads safely, we return `Arc::new(Mutex::new(dbl))`, `Arc::new(Mutex::new(trpl))`, and `Arc::new(Mutex::new(all_token_list))` instead.

To use the concurrent maps version, you do not have to add any flags, it is run by default. In the future, we could try out a lock-free, eventually consistent, concurrent multi-value map, [evmap](https://github.com/jonhoo/evmap).

# Testing for correctness
We test our outputs using the [ECE459-A2-Comparison-Tool](https://github.com/chjon/ECE459-A2-Comparison-Tool) kindly provided by [Jonathan Chung](https://github.com/chjon). We first save the outputs from the original `dictionary_builder`, and the multithreaded solution using the concurrent HashMaps, `DashMap`s. We then run `python3 compare.py <FILE_1.txt> <FILE_2.txt> [TOLERANCE]`, with a tolerance of 2*`num_threads`. We observe no out of tolerance results, assuming the results from the original `dictionary_builder` was free from error.

# Testing for performance
We test our performance by running the sample `/data` provided with our compiled code, and timing the runs using `hyperfine`. For example, we test the `HealthApp.log` using `hyperfine --warmup 2 'cargo run --release -- --raw-healthapp data/HealthApp.log --to-parse "20171223-22:15:41:672|Step_StandReportReceiver|30002312|REPORT : 7028 5017 150539 240" --before "calculateAltitudeWithCache totalAltitude=240" --after "onStandStepChanged 3601" --cutoff 10'`. We observe significant improvements in runtimes. Using the `ecetesla3` server, we observe a speedup of approximately 93% from the original version (2.247s vs 30.620s for 10 runs).