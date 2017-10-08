+++
css = []
date = "2017-06-18T11:04:09+10:00"
description = "Useful tips for programming with golang channels."
draft = false
highlight = true
scripts = []
keywords = ["go", "channels", "concurrency", "tips", "golang", "goroutines"]
tags = ["go", "channels", "concurrency", "tips"]
title = "Go Channels"

+++

Channels and goroutines in golang takes a novel approach to composing 
concurrent programs.
Correct use of them leads us into writing more understandable 
concurrent code.

This is a compilation of little things that I find useful when using channels.

## Fine-grained channel types
Passing a reference to a channel as an argument to a goroutine is a 
common way to establish the communication link between them.
When doing so, specifying a fine-grained channel type, makes it easier 
to reason about go routines. 
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
Reading from a closed channel on the other hand has a more forgiving behavior.
However, this could sometimes create an adverse effect on goroutine's behavior. 
For example, consider the following event loop implementation.
Once the channel referenced by ```c``` is closed, calls to read from 
channel will return the default value for the channel's type 
(```''``` in the case below). 
Net result is that the goroutine is now in a tight loop.

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

We can correct this behavior by observing the second, boolean value returned by 
the read operation.

```
func eventLoop(c <-chan string) {
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

At this point, you may wonder why we could not simply check for ```''```
in ```s```. We should not forget that ```''``` is a perfectly valid value
to be sent over ```chan string```.

To be contd...
