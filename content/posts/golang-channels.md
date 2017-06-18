+++
css = []
date = "2017-06-18T11:04:09+10:00"
description = "Useful tips when programming with golang channels."
draft = false
highlight = true
scripts = []
tags = ["go", "channels", "concurrency", "tips"]
title = "Go Channels"

+++

Channels and goroutines in golang is a novel approach to composition of 
concurrent programs.
Correct use of them leads us into more understandable concurrent programs.
This is a compilation of little things that I find useful when using channels.

## Fine-grained channel types
Specifying fine-grained channel types makes it easier to reason about 
go routines. 
Consider two routines, producer and consumer communicating via a channel.

```
func producer(c chan<- string) {
  <-c // compile error
}

func consumer(c <-chan string) {
  c<- "a" // compile error
}

c := make(chan string)
go producer(c)
go consumer(c)
```

With fine-grained channel types in ```producer``` and ```consumer``` functions,
compiler will ensure that they can only either read or write to the channel.

## Reading from closed channels
An attempt to write to a closed channel results in a panic.
However, reading from a closed channel has a more forgiving behaviour.
Consequently, if we have an event loop written without a guard, 
the goroutine will end-up in a tight loop.

```
func eventLoop(c <-chan string) {
	for {
		s := <-c
		fmt.Printf("received %s\n", s)
	}
}

c := make(chan string)
go eventLoop(c)
c <- "hello" // prints hello
close(c) // puts eventLoop goroutine into a tight loop.
```

To fix this, we can rewrite the loop as below.

```
func eventLoop2(c <-chan string) {
	for {
		s, ok := <-c
		if !ok {
			fmt.Println("event loop exited")
			return
		}
		fmt.Printf("received %s\n", s)
	}
}
```

To be contd...
