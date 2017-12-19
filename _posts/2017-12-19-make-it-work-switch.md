---
title: Make it work Switch
layout: post
published: true
---

During some performance testing of an application a colleague and I had trouble processing more than 150 requests per second per instance. After much hair-pulling the bottleneck was identified as the [`ServicePointManager.RequestLimit`][servicepoint]. This global variable controls the number of parallel HTTP requests that a .NET app will make to a given domain. By default it is set to 2, throttling the load that an off-the-shelf .NET app can place on a remote server.

It isn't the first time that I've encountered this variable limiting an application. In fact it isn't even the first time it has caused problems in that specific application; however the previous mitigation of setting it to 2000 was accidentally lost during a partial re-write.

These kinds of gotchas with default configuration are referred to at work as "make it work switches". The `ServicePointManager` is far from the only culprit. Software is littered wit them, such as Elastic Search's [`discovery.zen.ping.multicast.enabled`][es]; Javascript's [`"use strict";`][usestrict]; and the new `.csproj` format's [`<LangVersion>Latest</LangVersion>`][cslangversion]. These kind of papercut issues can really impact user experience. Not only are they annoying to keep track of, but are often stumbling blocks for beginners.

These kinds of stumbling blocks are often the most important to work on. They affect how software is perceived by new users, and how more experienced users feel about using your software. Feel is important. It can make super-fast software feel clunky and cumbersome to use. On the flip side a well designed interface which "just works" can make even the most complex things seem simple and intuitive.

Things that "just work" by default is often touted as the reason to use *macOS* or *iOS*. There's a reason for this. Apple has historically had a dedication to simplicity. Maybe too much so for some people. One of the ways that this often shows is the _lack_ of configuration available. The legend goes that if you wanted to add a config option Steve Jobs would ask you what the default should be. When you'd decided that then he would tell you to hard code it to that value. While configurable software allows you to quickly adapt it to do new and different things it does come at a cost. After all, there would be no *make it work* switches if there were no switches at all.

 [servicepoint]: https://docs.microsoft.com/en-gb/dotnet/api/system.net.servicepoint.connectionlimit?view=netcore-2.0#System_Net_ServicePoint_ConnectionLimit
 [es]: https://www.elastic.co/guide/en/elasticsearch/guide/1.x/_important_configuration_changes.html
 [usestrict]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode
 [cslangversion]: https://github.com/dotnet/docs/issues/2821#issuecomment-323646955