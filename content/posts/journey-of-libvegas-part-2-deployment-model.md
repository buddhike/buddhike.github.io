+++
date = "2024-02-26T09:00:00+11:00"
title = "Journey of libvegas (Part 2) - Deployment Model"
description = "Journey of libvegas (Part 2) - Summary of pros and cons considered when choosing the deployment model for libvegas"
draft = "false"
highlight = "true"
keywords = ["aws", "kinesis", "client", "streaming", "go"]
tags = ["aws", "kinesis", "client", "streaming", "go"]
+++

Interesting thing about stream storage clients like `libvegas` is that they pack
a lot of functionality. Therefore it's important to choose a programming
language suitable for this task. For us one of the key considerations for
choosing the appropriate programming language was the deployment model for
`libvegas`. Below we present the two models we investigated.

### Out of process deployment
In this model, we package `libvegas` as an executable and distribute it as a
part of the
library for each programming language. When users invoke Producer
and Consumer APIs, the language binding code will run the executable and
interact with send/receive functionality via some sort of an IPC channel.

![Out of process deployment model of libvegas](/journey-of-libvegas-part-2-deployment-model/out-of-process-deployment-model.png)

With this model, we can develop `libvegas` virtually in any programming language
we are familiar with. Since the language binding is just IPC code, it can also
be developed and tested relatively easily.

We can package this executable and all of its dependencies as single cohesive unit
and be confident about the compatibility in various operating environments.

Another benefit of running `libvegas` as an out of process executable is that we can
control the resources allocated for the child process more precisely.

Out of process model also presents some challenges as well. Users of the
executable now have to package any runtime dependencies required by `libvegas`
executable. For example, if we develop `libvegas` in `JavaScript`, users 
have to package a compatible `nodejs` with their application.

Another downside is, when users of `libvegas` setting up monitoring tools
for their client application, they have to take the child process into
consideration. This is an implementation detail that may not be obvious for many
who are used to importing a package from their desired packaged manager. More
importantly, users may not be familiar with specifics of monitoring relevant to
`libvegas`. Imagine `libvegas` developed in `C#` and used by a Go user. To
effectively monitor their application as a whole, Go user
has to be familiar with metrics such as `Gen x Collections` which are specific
to `.Net` framework.

### In process deployment
In process deployment model packages `libvegas` as a shared library. These
libraries are generally distributed as `.so`, `.dylib` and `.dll` files in `linux`,
`osx` and `windows` operating environments. 

![In process deployment model of libvegas](/journey-of-libvegas-part-2-deployment-model/in-process-deployment-model.png)

In this model, `libvegas` exposes its functionality as a `C` API and langugage
bindings will use their [Foreign Function Interface (FFI)](https://en.wikipedia.org/wiki/Foreign_function_interface) capabilities to invoke
it. In many ways this is very similar to how we access the functions of the
operating system from various programming languages.

In this model, `libvegas` is loaded into client's address space. With a careful
design we should be able to make the library integrate nicely with monitoring
and logging frameworks used to develop the client itself.

From user experience point of view, workflow for importing and deploying
`libvegas` would be similar to how one would use other shared libraries.

With in process deployment model, developing the language binding is a non trivial task. We have to
marshal data for `FFI` using facilities provided by the language. Sometimes
built-in mechanisms are too cumbersome and we would have to carefully choose a
package which abstracts some of that heavy lifting. Good example of this is
`Python`'s `FFI`. We can start with
[`ctypes`](https://docs.python.org/3.8/library/ctypes.html) but as the
complexity of our integration grows we may have to consider [one of many options](https://realpython.com/python-bindings-overview/#python-bindings-overview)
available to simplify this task. 

### Conclusion
Having considered both of these options, we concluded that we would choose the
in process deployment model. Whilst this may increase the complexity of
developing language bindings, it brings the first class experience to the end
user to develop and run `libvegas` based client applications in production.

From integration point of view, we would like our users to use their respective
package manager to acquire `libvegas` and follow a consistent approach for
everything else (packaging, deploying, logging and monitoring) when developing
their clients.
