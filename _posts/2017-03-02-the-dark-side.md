---
title: The Dark Side
layout: post
published: true
---

At work I'm paid to write C#. At home I'm a dedicated Mac user. When I initially learnt C# for the job interview years back I had to use [Mono](http://www.mono-project.com) to run it on my Mac. A lot has changed in the .NET world since then, most notably the arrival of [.NET Core](https://www.microsoft.com/net/core). So I finally decided to give it a try.

The installation on OS X was really simple. So far I'm really enjoying compiling stuff from the command line with the `dotnet` tool. `Project.json` files look to be a lot easier to branch and merge than `.csproj` files. Impressed with the whole thing I decided to knock together a quick project to give the whole thing a proper shakedown.

So far building and testing from emacs using `dotnet build` and `dotnet test` has been pretty good. There's a few rough corners which show themselves, such as `dotnet restore` working from the project root, `dotnet build` requiring a glob to find project files to build and `dotnet test` not accepting globs at all. I guess the tooling is still a preview after all. That said I did manage to get something working pretty easily and quickly. The result was [PollyTick](http://github.com/iwillspeak/pollytick); a library for collecting execution statistics for executions of Polly policies. Polly itself is a great little library, but that's for another day.

Setting up a build on Appveyor for this library was pretty simple too. I've previously [used Appveyor to build Rust on Windows](https://github.com/rust-onig/rust-onig/blob/master/appveyor.yml). The whole .NET Core restore, build, test, package workflow fits quite nicely into Appveyor's four stages of workflow. After [a bit of fiddling](https://github.com/iwillspeak/PollyTick/pull/1) I managed to get a build up and running. The resulting config ended up being pretty simple:

{% highlight yaml %}
image: Visual Studio 2015
install:
  - ps: dotnet --version
  - ps: dotnet restore
build_script:
  - ps: dotnet build src/*/project.json
test_script:
  - ps: ForEach ($folder in (Get-ChildItem -Path test -Directory)) { dotnet test $folder.FullName }
{% endhighlight %}

The fiddly bit was being able to run all test projects. The `dotnet test` command doesn't support the globs that `dotnet build` does, so a bit of PowerShell is required to get the job done.  I did [experiment with caching NuGet packages](https://github.com/iwillspeak/PollyTick/commit/63646855a70b5ae42ada37531c6c79174bc7afae) however that just seemed to make the build slower. Overall the process was pretty painless though. The Appveyor builds themselves seem super responsive and it builds this little project super quickly.

Here's to the future of .NET and C#.
