# CS 389 HW 05 - Lies, Statistics, and Benchmarks
### Names : Prasun and Hrishee

## New File Overview
`latency.csv` : CSV file containing our latency data for part 2.

`latency.png` : Cumulative frequency graph of the latency data in the CSV file.

`hw_5_systems.zip` : zip file submission of our repository. Refer to our [README for HW4](https://github.com/prg007/Lets_Network) for why we need a zip file.
                Inside the `zip` file we have:

We'll go over the new files inside the zip file (we elide descriptions of anything from HW4, HW3, and HW2, since those files are the same.):

`driver.cc` : Our implementation of the benchmark program. This includes the workload generation and client driver.


## Getting Started
Please follow the instructions of how to set up and compile in our [README for HW4](https://github.com/prg007/Lets_Network).
The compilation instructions are identical.

### How to run the benchmark
After compiling, run the server with `./server`, using any custom options as mentioned in out [README for HW4](https://github.com/prg007/Lets_Network).
Then in a separate terminal window, to run the benchmark, use the command

`./driver nreq CAPACITY NUM_REQUESTS NUM_UNIQUE_KEYS NUM_UNIQUE_VALS GET_RATIO SET_RATIO DEL_RATIO`

After it has finished running, it should report the hit rate, the 95th percentile latency, and the mean throughput, as well
as writing the data out to the CSV file.

The meaning of the driver parameters above and our default values for running them (rationale explained
throughout the rest of this writeup) are given in the main function:

Parameter name | Default Value
----------|------------
`nreq` : are number of requests to be randomly selected for measurement (must be less than NUM_REQUESTS) | 7000
`CAPACITY` : maxmem of server running | 1000000
`NUM_REQUESTS` : number of total requests to send to the server | 15000
`NUM_UNIQUE_KEYS`: number of possible unique keys to sample from in workload generation (called the key pool) | 100
`NUM_UNIQUE_VALS`: number of possible unique values to sample from for workload generation (called the value pool)| 100
`GET_RATIO` : Integer that defines GET_RATIO : SET_RATIO : DEL_RATIO, e.g. (30 : 1 : 15  for the ETC ratio) | 30
`SET_RATIO` : see above | 1
`DEL_RATIO` : see above | 15

In other words, if `GET_RATIO = g`, `SET_RATIO = s`, and `DEL_RATIO = d`, then `g : s : d` means that out of `NUM_REQUESTS`
requests, g/(g+s+d) is the fraction of get requests, s/(g+s+d) is the fraction of set requests, and d/(g+s+d) is the fraction
of del requests.

Note: sometimes you may get a "Cannot connect to server. Exiting". If so, you either have to rerun
or `NUM_REQUESTS` or `CAPACITY`/ server `maxmem` may be too high. (In practice, we noticed that any higher
than around 15000 requests and 1000000 bytes of cache server caused this crash. This was a limitation
in collecting accurate data, but we worked with what we had).

## Workload Generation

### Trying to emulate the ETC Workload
To emulate the ETC workload to a feasible degree, we focused on four aspects: composition of requests, key sizes,
value sizes, and key reuse. Keys were of type `std::string` comprised of random alphabetic characters. Similarly
for values, except they were of type `const char*`.

#### Composition of Requests
We initially started with a ratio of 30 : 1 : 15 of get to set to del requests to imitate the ETC workload.
However, we allowed for the option to vary the ratio through command line parameters in case we needed to. Spoiler,
we did.


#### Key sizes
We generated a random pool of keys (the key pool) to be used for generating the workload. To imitate ETC's
workload, we made sure that 70% of keys were 40 bytes and below, while the remaining 30% were between 41 and
80 bytes. This is approximately the behavior of ETC's workload as described in figure 2, graph 1 of the paper.


#### Value sizes
To generate the value pool in accordance with what the paper said, 50 percent of our values had sizes in the range
of 1 to 10 bytes and the remaining 50 percent of our values had sizes in the range of 11 to 500 bytes. While the paper
has the upper bound to be 1000000 bytes, we truncated it because we are not gods.


#### Key Reuse
The paper mentioned that 50% of the keys in the key pool occur in only 1% of requests. We imitated this by
factoring it into our workload generation, meaning that of the `NUM_REQUESTS` requests generated, about
99% of them contain half of keys while the other 1% of them contain the other half.


### Warming up the cache
To warm up the cache before executing the workload, we made a number of set requests (namely `CAPACITY`/1000) using
uniformly random key value pairs pulled from the key and value pools.  Since the value sizes vary from 1 byte to 500 bytes
this may not mean that the cache is boiling hot (i.e. full), but it at least ensures
an acceptable level of warmth (we call it "lukewarmth"). The randomization comes with a risk as well, since if all the warming
set requests used large values, then there would not be many key-value pairs in the cache. However, in terms of data
fidelity, it is better than having a fixed warm up routine that guaranteeably fills the cache.

### Tuning the hitrate
To get a higher hitrate while maintaining around a 2/3 ratio between the number of get requests to `NUM_REQUESTS`,
we heuristically lowered the delete ratio while increasing the set ratio. In theory, this ensures fewer invalidation misses.
With the get, set, and del ratio being set to 50 : 22 : 1 and maxmem set to 1000000, we discovered our best possible hit rate to be 0.66.
There's a good chance that there could be a better set of ratios which yields a hit rate closer to 0.8, but we couldn't find such
a setting (while keeping GETs to around 2/3 of NUM_REQUESTS, at least).

## Baseline latency and throughput

Our `baseline_latencies` and `baseline_performance` are part of `driver.cc`. The data we obtained was from
running

`./driver 7000 1000000 150000 100 100 50 22 1`

with both the server and client running locally. We would have liked to sample over a higher number of requests, but as mentioned previously our server would crash
if it exceeded more than around 15000 requests. This is probably due to a mixture of incompetence on our part and quirks of the
disparaged client/server frameworks (cpr and Crow) we used. Recall `nreq = 7000` means that out of 15000 total requests, we
randomly measure 7000 of them. We chose 7000 because it struck a balance between
not accounting for every request that came in while still having somewhat interpretable data.
Nevertheless, we obtained a somewhat polished distribution:

![Graph for Latency Part 2](https://github.com/prg007/HW5---Network-Benchmark/blob/master/latency.png)

In the above graph, the x-axis is the latency (ms) while the y-axis
is the cumulative frequency. In other words, for any point x, the corresponding y value is the number of latencies
less than or equal to x present in the data. We obtained 95th percentile latency of 1.013 ms and a mean throughput of
13133.2 requests/sec.


## Sensitivity Testing

Our four parameters for sensitivity testing are as follows (while keeping other parameters constant):
- Size of maximum cache memory
- Compiler and/or compilation options (O3, O2, O0)
- Changing the Load factor
- Changing the number of Unique keys in the key pool

We noticed a good deal of fluctuation, possibly due to the fact that we are running only 15000 requests. However,
as described by the above limitations, we made with what we had. Each data point is averaged over 5 trials.

Size of maximum cache memory | 95th percentile (ms)| Throughput (reqs/sec) | Hit Rate
-------------|----------------|----------------|---------------
100 | 1.056 | 9655.17 | 0.646
1000 | 1.117 | 9421.27 |  0.641
10000 | 0.998 | 10869.6 | 0.651
100000 | 1.003 | 10233.9 | 0.65
1000000 | 1.013 | 13133.2 | 0.66

The memached paper mentioned that as maxmem increases, the hitrate should increase as well.
We see only a modest hitrate increase here, even though our maxmem values are increasing by an
order of magnitude each time. Perhaps this is because we only performed 15000 requests.

If maxmem increases, then we would expect higher hitrates (as we nominally have here). Thus on average,
there are more cache hits than misses comparatively. In our case, cache hits are more expensive than cache misses
(see the discussion of the fourth table as to why we think so), so the 95th percentile time increases as well.

However, the slower times are also nominal increases, which when combined with increasing throughput, may suggest there is
a confounding factor (one we do not know about) present.

Compiler Optimization | 95th percentile (ms)| Throughput (reqs/sec) | Hit Rate
-------------|----------------|----------------|---------------
-O3| 1.013| 13133.2 | 0.66
-O2 | 1.131 | 5604.48 | 0.657
-O0 | 1.719 | 2105.26 | 0.656

An intuitive explanation is that compiler optimizations yield faster code, which is reflected
in faster 95th percentile times and higher throughput. Note that we compiled the `server.cc` and
`cache_client.cc` at the same time as the driver, so it may have been that optimizations were applied not
only to the benchmark but to the actual server code themselves.


Load Factor | 95th percentile (ms) | Throughput (reqs/sec) | Hit Rate
-------------|----------------|----------------|---------------
0.25| 1.222| 4350.53 | 0.656
0.50 | 1.244 | 7543.1 | 0.6524
0.75 | 1.013 | 13133.2 | 0.66
1.00 | 0.552 | 35353.5 | 0.654

With a higher load factor, rehashing in the `std::unordered_map` takes place less often,
so we would expect less computation to occur, allowing us to pass more requests per second. Even though
a higher load factor means that on average more k-v pairs exist in each bucket (making search through
the bucket slower), rehashing is still a more expensive operation.
We see some evidence of this here, with the throughput increasing as the load factor increases.

Number of Unique Keys | 95th percentile (ms) | Throughput (reqs/sec) | Hit Rate
-------------|----------------|---------------|-------
10| 1.26| 4350.53 | 0.66
100 | 1.013 | 7543.1 | 0.66
1000 | 0.651 | 13133.2 | 0.44
10000 | 0.59 | 35353.5 | 0.27


We see that more possible keys to pull from is related to
faster 95th percentile times and higher throughput, but lower hit rates. This makes some sense, as 
more keys to choose from decreases the probability of having a specific key in the cache (i.e. a cache miss).
In memcached, a cache miss was more expensive than a cache hit because the server had to go grab data from a database.
In our implementation, we have no such database. Nevertheless, a cache miss is more expensive than a cache hit in our case as well,
because in our server implementation, we perform a linear search for the key then return `nullptr` if the key is not in the cache;
if the key is in the cache, we perform a linear search then perform the added computation to return the key.
Therefore more possible keys implies more cache misses, which finally implies faster times.

