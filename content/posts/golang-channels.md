+++
css = []
date = "2017-06-18T11:04:09+10:00"
description = ""
draft = false
highlight = true
scripts = []
tags = []
title = "Go Channels"

+++

Channels and go routines in golang is a novel approach to composition of 
concurrent programs.
Correct use of them make it a lot easier to reason about concurrent
programs.
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

To be contd...
