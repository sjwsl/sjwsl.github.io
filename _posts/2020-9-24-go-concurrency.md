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

## The Go approach

Don't communicate by sharing memory, share memory by communicating. 

~~lock, mutex, conditional variables~~ $\rightarrow$ **channels**

## Concurrency patterns



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