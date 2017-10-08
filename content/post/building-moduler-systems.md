+++
date = "2017-10-07T09:28:47+11:00"
title = "Building Modular Systems"
draft = "true"
+++

Large systems are usually constructed by composing many small systems. My favorite 
browser for example distributes its work across different processes while
I'm conveniently enjoying my morning dose of hacker news on one screen. ![Chromium task manager](/img/chromium-task-manager.png)

This also applies to server side systems with the added complexity of 
modules being distributed across multiple machines, built using different 
tool-chains etc.

Modularisation is great as it gives us the ability to reduce coupling and 
cohesion. However, ultimately we are only interested in the overall 
state of the system rather than the individual modules. Consider a system of 
two modules `A` and `B` for instance. The overall state of this system is a set 
`{A1, B1}` where `A1, B1` are versions of `A` and `B`. If `A` and `B`
both had multiple version choices and system is composed by choosing 2, how 
many combinations we can make? There's a [formula](http://mathworld.wolfram.com/BinomialCoefficient.html) for that if we 
want to knock ourself out. The point is,
the more modules we have and the number of versions we create over time, 
we have a large number of possible combinations for our system.

Tracking of these combinations gives us the ability to reason about how 
the system changes over time. For this type of tracking we use source control 
systems but there are two distinct patterns in terms of how they are organised.

- All modules are stored in one source repository
- Each module is maintained in its own repository

Both approaches require some tooling to establish the state of the system.
Chromium for example consists of multiple independent modules 
such as [V8](https://chromium.googlesource.com/v8/v8.git) and [webrtc](https://webrtc.googlesource.com/src.git) uses [depot_tools](https://chromium.googlesource.com/chromium/tools/depot_tools/+/master/fetch.py) to archive this.    
 
For the rest of this post our focus is going to be on [mbt](https://github.com/buddyspike/mbt) which 
is a tool for building modular systems stored in a single git repository.

## Design goals
mbt is designed as a tool to orchestrate existing build tools rather than
an alternative to a build runner like `make`, `msbuild` or `maven`. Below we 
discuss a few design goals of mbt.

### Integrate with current build utilities
mbt is a cli tool that can be used locally or inside current build systems like [Jenkins](https://jenkins.io), [Teamcity](https://www.jetbrains.com/teamcity/).

### Declarative 
  
Much like [Travis](https://travis-ci.org) and [AppVeyor](https://www.appveyor.com/), mbt 
discovers your modules by a spec file you place in the root of each module directory.


### Fast
### Consistent

### Support all types of automated deployments
### Optimise the build by building only what's changed 