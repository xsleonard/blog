+++
title = "Avoiding Locks in Golang"
date = "2014-01-15T00:00:00+00:00"
tags = ["software", "go", "golang", "concurrency"]
+++

[Golang maps are not safe for concurrent writes](http://golang.org/doc/faq#atomic_maps).

Common solutions are to use `sync.Mutex`, `sync.RWMutex`, or a mutex implemented with `chan`.  Depending on how your application is structured, we can avoid locking with `select`.

### Synchronizing map access with select

```go
package main

import (
    "math/rand"
    "time"
)

var m = make(map[int]int)

// Writes a random k,v pair to the map
func write() {
    k := rand.Int()
    m[k] = rand.Int()
}

// Returns a random value stored in the map
func read() int {
    keys := make([]int, 0, len(m))
    for k, _ := range m {
        keys = append(keys, k)
    }
    key := keys[rand.Int()%len(keys)]
    return m[key]
}

func main() {
    m[0] = 0 // avoids 0 len map
    quit := time.After(time.Second)
    r := time.Tick(time.Millisecond)
    w := time.Tick(time.Millisecond * 2)
loop:
    for {
        select {
        case <-r:
            read()
        case <-w:
            write()
        case <-quit:
            break loop
        }
    }
}
```

In this example, if `m` were automatically safe for concurrent RW, we would still need manual locking for the body of `read()` due to the `len(m)` call.

This method is unsuitable if the functions called in any of the `case:` take significant execution time relative to how often a channel in the `select` has data.  Channel queues would fill up and timers would be delayed or skipped.

******

Benchmarking
------------

I've created a repo for [comparing the performance of using select versus sync.Mutex](https://github.com/xsleonard/select-vs-mutex).

```sh
git clone https://github.com/xsleonard/select-vs-mutex.git
cd select-vs-mutex/
go test -bench=With
```

<table class="table table-striped">
<thead><td>Name</td><td>Runs</td><td>Time</td></thead>
<tbody>
<tr><td>BenchmarkWithLocks10us</td><td>500</td><td>7151996 ns/op</td></tr>
<tr><td>BenchmarkWithoutLocks10us</td><td>100</td><td>11079513 ns/op</td></tr>
<tr><td>BenchmarkWithLocks100us</td><td>100</td><td>20253715 ns/op</td></tr>
<tr><td>BenchmarkWithoutLocks100us</td><td>100</td><td>21677902 ns/op</td></tr>
<tr><td>BenchmarkWithLocks1000us</td><td>10</td><td>200301300 ns/op</td></tr>
<tr><td>BenchmarkWithoutLocks1000us</td><td>10</td><td>200339458 ns/op</td></tr></tbody>
</table>

The `100us` suffix is how often the read ticker fires.  The write ticker fires twice as often as the read ticker.

The locking method is significantly faster with a high tick rate, but performance converges with a 1ms tick rate.  The relative poor performance at 10us may be due our `write()` or `read()` methods taking longer than the tick rate, or limitations in the go runtime handling blocking `select` with such a short interval.
