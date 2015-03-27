---
title: 'Microcrates: Little Flecks of Rust'
layout: post
published: true
---

Gists are really just git repos with no directory structure. This is something that some Rubyists have taken advantage of in the past to create ["Microgems"](http://jeffkreeftmeijer.com/2011/microgems-five-minute-rubygems/). Would it be possible to do the same thing in Rust?

I was in the process of messing around with some Rust code. I had created a new crate with cargo and hacked around a bit. Not wanting to lose my changes I'd created a git repository too. We aren't talking about much code here, just a few dozen lines in a single `.rs` file. I didn't really want to create a full GitHub repo just for this small amount of code. "Wouldn't it be really nice if I could just put it all in a Gist" I thought.

By convention a Rust crate has a few different directories, `src/` for all of the source files, `test/` for integration tests and so on; at the top level is the `Cargo.tom` file containing the package metadata. We can't use these in a Gist. We need to just stick our source files right next to the `Cargo.tom`. It turns out that configuring Cargo to understand our wacky new folder structure is actually quite simple.

{% highlight toml %}
[package]
 
name = "..."
version = "0.0.1"
authors = ["..."]
 
[[bin]]
    name = "example"
    path = "main.rs"
{% endhighlight %}

If you're creating a library rather than an executable you'll ned to use the `[lib]` section instead of the `[[bin]]` array. There's more information about what you can configure for each target [in the cargo docs][cargo_target]. You can see the results in action in [a simple hand-written lexer I wrote].

  [cargo_target]: http://doc.crates.io/manifest.html#configuring-a-target