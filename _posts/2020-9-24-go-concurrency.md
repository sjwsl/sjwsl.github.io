---
title: Go Concurrency
categories: 
    - Essay
tag:
    - Golang
---

Concurrency is the key to designing high performance network services. Go's concurrency primitives (goroutines and channels) provide a simple and efficient means of expressing concurrent execution.

[The Go Memory Model](https://golang.org/ref/mem)

[Google I/O 2012 - Go Concurrency Patterns](https://www.youtube.com/watch?v=f6kdp27TYZs)

## Channel: the Go approach

Don't communicate by sharing memory, share memory by communicating. 

~~lock, mutex, conditional variables~~ $\rightarrow$ **channel**

```go 
chan   // read-write
<-chan // read only
chan<- // write only
```

## Concurrency patterns

### Generator: function that returns a channel

channel as service

```go
func Generator(msg string) <-chan string {
    c := make(chan string)
    go func() {
        for i := 0; ; i++ {
            c <- fmt.Sprintf("%s: %d", msg, i)
            time.Sleep(time.Second)
        }
    }()
    return c
}

c := Generator("Bob")
for i := 0; i < 5; i++ {
    fmt.Printf("%s", <-c)
}
```

### Timeout

Timeout for a single execute

```go
c := Generator("Bob")
for {
    select {
    case s:= <-c:
        fmt.Println(s)
    case <-time.After(time.Second):
        return
    }
}
```

Timeout for whole conversation

```go
c := Generator("Bob")
timeout := time.After(time.Second)
for {
    select {
    case s:= <-c:
        fmt.Println(s)
    case <-timeout:
        return
    }
}
```

`time.After` is also a Generator like

```go
func After(t time.Duration) <-chan bool {
    c := make(chan bool)
    go func() {
        time.Sleep(t)
        c <- true 
    }
    return c
}
```

Note that the goroutine may be blocked on `c <- true` forever if no one receive it, but it will be Garbage-collected Automatically.

### Multiplexing: let whosoever is ready talk

```go
func funIn(input1, input2 <-chan string) <-chan string {
    c := make(chan string)
    go func() {
        select{
        case v1 := <-input1:
            c <- v1
        case v2 := <-input2:
            c <- v2
        }
    }
    return c
}
```

### Quit channel: tell a Generator to stop

```go
func Generator(msg string, quit <-chan bool) <-chan string {
    c := make(chan string)
    go func() {
        for i := 0; ; i++ {
            select {
                case c <- fmt.Sprintf("%s: %d", msg, i):
                    time.Sleep(time.Second)
                case <-quit:
                    return
            }
        }
    }()
    return c
}

quit := make(chan bool)
c := Generator("Bob", quit)
for i := 0; i < 10; i++ {
    fmt.Println(<-c)
}
quit <- true
```

Two-way waiting

```go
quit <- true
<-quit

...
    select {
    case c <- fmt.Sprintf("%s: %d", msg, i):
        time.Sleep(time.Second)
    case <-quit:
        // do something like GC
        quit <- true
        return
    }
...
```


## Incorrect synchronization

Compilers and processors may reorder the reads and writes executed within a single goroutine when the reordering does not change the behavior within that goroutine as defined by the language specification. Because of this reordering, **the execution order observed by one goroutine may differ from the order perceived by another.**

If we have `a = 1; b = 2;` in one goroutine, maybe another goroutine observe the updated value of `b` before `a`. This fact invalidates a few common idioms.

```go
var a string
var done bool

func setup() {
	a = "hello, world"
	done = true
}

func main() {
	go setup()
	for !done {
	}
	print(a)
}
```

As before, there is no guarantee that, in main, **observing the write to done implies observing the write to a**, so this program could print an empty string too. Worse, **there is no guarantee that the write to done will ever be observed by main**, since there are no synchronization events between the two threads. The loop in main is not guaranteed to finish.

The solution is: **use explicit synchronization.**