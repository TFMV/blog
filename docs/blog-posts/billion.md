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

## Optimizing for Speed: The Real Fun Begins

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

## The Final Results

After all these optimizations, here's how fast I managed to get it:

```bash
File opened with mmap in: 40.292µs
File size: 8793.81 MB
Using 10 CPU cores
Processing file in 1100 batches of ~8 MB each
Workers initialized in: 17.625µs
Reading complete: 1100 batches in 14.716501416s
All 1100 result batches processed in 14.777454417s
Output formatted for 180 stations in: 128.375µs
Total processing time: 14.777804084s
Challenge completed in: 14.792802708s
```

**14.79 seconds for a billion rows.**

Not bad.

## Lessons Learned

- **Memory mapping is a game-changer.** Don't read line-by-line if you can avoid it.
- **Goroutines help, but synchronization is key.** If you're not careful, concurrency overhead will eat all your gains.
- **String parsing is a hidden bottleneck.** Avoid unnecessary allocations wherever possible.
- **Hash tables can make or break performance.** Go's built-in map is great, but custom solutions can be faster for specialized use cases.
