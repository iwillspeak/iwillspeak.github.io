---
group: blog
title: Testing Testing Testing
layout: post
---

Scripting languages are all the rage these days. I am a fan of Python, although you may prefer Ruby, Node.js or whatever it is the cool kids are using these days. The exact language that takes your fancy however doesn't really matter. They almost all have one thing in common: Unit tests. This is not a startling realisation, nor is it restricted to the realms of modern scripting languages. I hear that even the assembly code running on the jet engine of your favourite airliner is unit tested (a reassuring fact). Startling or not what is important about unit testing with scripting languages is the ease at which the development cycle progresses. If only it could be as easy for other languages.

This ease of dynamic language test driven development is largely to do with automated test runners, such as [nosy](https://bitbucket.org/douglatornell/nosy) for Python. These tools are great, however I spend quite a lot of time programming in languages such as C and C++. I had to settle for manually running the test suite every time I made the change. 

There is a limit on the number of times a programmer can repeat a task before he automates it. When I reached this limit I decided to introduce myself to Ruby, and [Snooper](http://github.com/iwillspeak/snooper/) was born.

Snooper is a lightweight test automation tool, it monitors files and folders while you work and re-runs your tests when you change something. Snooper doesn’t care what language you’re using or what framework you are testing with, it’s all configureable.

![Snooper runs some Tests]({{ site.url }}/img/posts/snooper_test_output.jpg)

All snooper does is watch the filesystem for changes and runs a command when a given set of files changes. Dependant on the success or failure of the command a prettified status bar is printed, with plenty of colour to ensure that you can see the test status from far off. It is a small set of features, but a neat one (even if I do say so myself).

I hope you find it as useful as I do. Feel free to [tweet]({{ site.twitter_url }}) me any cool uses of Snooper. Until next time, keep on testing!
