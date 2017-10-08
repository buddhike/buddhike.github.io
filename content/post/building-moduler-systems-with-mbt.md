+++
date = "2017-10-07T09:28:47+11:00"
title = "Building Modular Systems with mbt"
draft = "false"
highlight = "true"
keywords = ["mbt", "build", "dev-ops", "monorepo"]
tags = ["mbt", "build", "dev-ops", "monorepo"]
+++

Large systems are usually constructed by composing many small systems. My favorite 
browser for example distributes its work across different processes while
I'm conveniently enjoying my morning dose of hacker news. ![Chromium task manager](/img/chromium-task-manager.png)

This also applies to server side systems with the added complexity of 
modules being distributed across multiple machines, built using different 
tool-chains etc.

Modularisation is great as it gives us the ability to reduce coupling and 
cohesion. However, ultimately, we are only interested in the overall 
state of the system rather than the individual modules. Therefore it is worth
exploring what we consider as the overall system state.

Imagine a system of two modules `A` and `B` for instance. 
The overall state of this system is a set 
`{A1, B1}` where `A1, B1` are versions of `A` and `B`. If `A` and `B`
both had multiple version choices and system is composed by choosing 2, how 
many combinations we can make? There's a [formula](http://mathworld.wolfram.com/BinomialCoefficient.html) for that if we 
want to be sure. The point is, the number of combinations that make up the system
increases with the number of modules and their versions.

Tracking these combinations gives us the ability to reason about how 
the system changes over time. Nowadays, for this type of tracking we 
use source control systems but there are two distinct patterns in 
terms of how they are organised.

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
Spec file must be called `.mbt.yml` and the content instructing how to build your 
module is formatted in [yaml](http://yaml.org/spec/).

For example consider the following `.mbt.yml` file.

```yaml
name: my-app
build:
  windows:
    cmd: powershell
    args: [-ExecutionPolicy, Bypass, -File, .\build.ps1]
  linux:
    cmd: ./build.sh
```

With this file committed to your repository, you can trigger the build for 
the master branch by running the following mbt command.

```sh
mbt build branch master --in [path to repo]
```

When the preceding command is executed in a windows host, it would run `build.ps1` 
bypassing the execution policy. On a linux host, it would run `build.sh` file.

With this approach mbt gives you the complete control over what to be done 
when build is executed. 

### Fast
When a repository contains only a single module, build is triggered by changes to 
that module. However, when multiple modules are stored together, if a change to 
one module triggers the build of all modules, build system will not be scalable. 

mbt provides options to execute the builds only for the changed modules. It works 
out the changes automatically by integrating with git.

```sh
# build the diff of two branches
mbt build pr --src [source branch name] --dst [destination branch name] --in [path to repo] 

# build the diff of two commits
mbt build diff --from [commit sha] --to [commit sha] --in [path to repo]

```

### Consistent
Tagging the builds with the git commit sha or build runner's build number 
is a typical approach when building single module repositories. 
For multiple module repositories however, 
if we are not intending to build all modules for each commit, commit sha would 
not be representative build tag.

To solve this problem, when building the source tree for a particular commit,
mbt calculates a SHA1 hash for each module based on the content of module directory stored 
in git tree. This gives a consistent version tag across multiple commits for modules that haven't been changed.

The mechanics of this hash calculation is similar to 
how git tree objects are hashed [internally](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects). 

Among other module specific attributes, the version tag calculated for 
each module is available to your build tools via `MBT_APP_VERSION` 
environment variable. 

### Template processing

When a build is executed by mbt, it builds up metadata about modules in your repository.
The metadata consists of the module version number and attributes you specified in `.mbt.yml` file.
At this point it's also worth noting that `.mbt.yml` allows you to store arbitrary data structure 
under its `properties` field as shown below.

```yaml
name: my-app
build:
  windows:
    cmd: powershell
    args: [-ExecutionPolicy, Bypass, -File, .\build.ps1]
  linux:
    cmd: ./build.sh
properties:
  type: spa     # these key-value pairs can be anything
  path: my-app
  nested:       # they can also be nested multiple times
    foo: bar    
```

mbt exposes its metadata system to templates you write in [go](https://golang.org/pkg/text/template/).

For instance, the following template can be used to generate a text document 
with all modules and their version.

```go
{{range $app := .Applications}}
{{$app.Name}} {{$app.Version}}
{{end}}
```

Provided the template is stored in your repository, you can run the following command 
generate a list of modules and their version in master branch.

```sh
mbt apply branch master --to [path to template] --in [path to repo]
```

A text file with a list of modules and their version might be of little use. But the same
technique can be used to generate more useful artifacts such as

- Custom deployment scripts (e.g. Powershell scripts that configures modules in IIS)
- nginx configs
- AWS Cloud Formation Templates
- Azure Resource Templates
- Kubernetes deployment files
 
### Wrap up
That's a quick tour of motivations behind mbt. 
It is still quite young and under a lot of churn. 
Do you think mbt would help you with a build problem you've been wrestling with lately?
