# The Billion Row Challenge

## A Fun Exercise in Speed

I came across the Billion Row Challenge the other day. Sounded fun. Take a massive dataset, process it as fast as possible, and see how well you can optimize your approach. Right up my alley.

The challenge itself is simple:

You get a file full of billions of weather station measurements, formatted as `station_name;temperature_value`.

The goal? Compute the min, mean, and max temperature for each station as fast as possible.

Seems straightforward, right? It never is.

## The First Attempt: The "Just Get It Working" Version

The naive approach is always the same:

1. Read the file line by line.
2. Split each line at the semicolon.
3. Parse the temperature.
4. Store the results in a map.
5. Print out the min, mean, and max for each station.

I wrote a basic Go program to do exactly that. It worked. It also took forever.

Reading a massive text file line by line? Slow. Parsing strings with `strings.Split`? Slow. Allocating memory every time a new station appears? Very slow.

Clearly, I needed something better.

## V1: First Round of Optimizations

At this point, I knew I had to rethink the whole approach. Here's what I ended up doing:

### 1. Memory Mapping the File

Instead of reading line by line, I used mmap (memory-mapped files) to treat the file as if it were already loaded in memory. This avoids expensive system calls and allows for ultra-fast access to data.

```go
reader, err := mmap.Open(filename)
```

This alone gave a huge speed boost.

### 2. Parallel Processing

Go has goroutines. I planned to use them.

- Split the file into chunks.
- Process each chunk in a separate worker.
- Aggregate the results at the end.

```go
numCPU := runtime.NumCPU()
for i := 0; i < numCPU; i++ {
    workerWg.Add(1)
    go Worker(i, jobs, results, &workerWg)
}
```

Each worker handled a chunk of data and sent its results back through a channel. This meant I could fully utilize my CPU cores instead of waiting around for I/O.

### 3. Custom Hash Table for Fast Lookups

Instead of using Go's built-in `map[string]Stats`, I wrote a custom hash table using open addressing with linear probing. This reduced memory overhead and improved lookup speed when updating station statistics.

```go
capacity := nextPowerOfTwo(expectedStations)
m.keys = make([]string, capacity)
m.values = make([]StationStats, capacity)
m.occupied = make([]bool, capacity)
```

Every lookup or insert happened in constant time.

### 4. Zero-Allocation String Parsing

Go's default string parsing methods allocate memory like there's no tomorrow. Instead of using `strings.Split`, I manually found the separator and used unsafe string conversions to avoid unnecessary allocations.

```go
sepIdx := bytes.IndexByte(line, ';')
station := UnsafeString(line[:sepIdx])
measurement, err := ParseFloat(line[sepIdx+1:])
```

No extra allocations. No garbage collection overhead.

## V1 Results: Getting Closer

After these optimizations, I got the processing time down to about 14.79 seconds for a billion rows. Not bad, but I knew we could do better.

## V2: Breaking the Speed Barrier

The journey to sub-4-second processing required even more aggressive optimizations:

### 1. Integer-Based Temperature Storage

Instead of using floats, we store temperatures as int16 with an implied decimal point. This not only saves memory but also makes calculations much faster.

### 2. Fixed-Size Buffers

We pre-allocate fixed-size buffers for station names, eliminating dynamic allocations during processing:

```go
type stationName struct {
    hashVal uint32
    byteLen int
    name    [MaxLineLength]byte
}
```

### 3. Larger Batch Sizes

We increased the batch size to 256MB, which significantly improved I/O throughput:

```go
const BatchSizeBytes = 256 * 1024 * 1024 // 256 MB
```

### 4. Branchless Temperature Parsing

We rewrote the temperature parsing to be as branchless as possible, making it more CPU-cache friendly.

### 5. Lock-Free Processing

By carefully designing our data structures, we eliminated the need for locks in our parallel processing pipeline.

## The Final Results: Breaking Records

The V2 implementation achieved something remarkable:

```bash
=== Benchmark Summary ===
Total Processing Time: 3.35 seconds
Throughput: 298.16 million rows/second
Memory Used: 516.91 MB
Number of CPUs: 10
Batch Size: 256 MB
=====================
```

**3.35 seconds for a billion rows.** That's a 77% improvement over V1!

## Lessons Learned

1. **Memory efficiency is king.** Fixed-size buffers and integer math made a huge difference.
2. **Batch size matters.** Larger batches meant better I/O throughput.
3. **Avoid branches.** CPU branch prediction is expensive; eliminate branches where possible.
4. **Know your hardware.** Understanding CPU cache lines and memory alignment paid off.
5. **Sometimes unsafe is necessary.** Strategic use of unsafe operations can yield significant performance gains.
6. **Measure, don't guess.** Every optimization was validated with benchmarks.

The journey from 14.79 seconds to 3.35 seconds shows that there's always room for optimization when you're willing to dig deep enough into the problem. It's not just about writing faster code—it's about understanding the entire stack, from the hardware up.
