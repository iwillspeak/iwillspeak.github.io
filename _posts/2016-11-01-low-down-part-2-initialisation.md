---
title: Low Down - Part 2 - Initialisation
layout: post
published: true
---

In my [previous post]({% post_url 2016-10-23-the-low-down-using-llvm-from-rust %}) I left out something quite important: initialising the components of LLVM we wanted to use. Luckily there’s a quick and easy way to get that sorted, and we get to learn about a neat part of the Rust standard library.

LLVM itself is composed of many modular libraries. This is great for library consumers as you can pick just the bits you want to use in whatever application you’re writing. It does however mean  there’s not a single `initialise` method to call. Some modules, such as the [PassRegistry][pass_registry_initialisation], expect you to initialise each object which is created. For compilation though we are expected to initialise the targets we want to use and, optionally, their ASM printers and parsers.

Some of the target initialisation routines can be called more than once, but not all are guaranteed to be reentrant or thread safe. To solve this we can use Rust’s [Once][rust_once] to ensure that we only call the initialisation routine once:

{% highlight rust %}
fn ensure_initialised() {
    use std::sync::{Once, ONCE_INIT};

    static INIT: Once = ONCE_INIT;

    INIT.call_once(|| {
        // Initialise LLVM
        unsafe {
            execution_engine::LLVMLinkInMCJIT();
            if target::LLVM_InitializeNativeTarget() != 0 {
                panic!("Could not initialise target");
            }
            if target::LLVM_InitializeNativeAsmPrinter() != 0 {
                panic!("Could not initialise ASM Printer");
            }
        }
    });
}
{% endhighlight %}

Here we are linking in the MC JIT so we can create and execute machine code for the current target, and initialising the current and it’s ASM printer. This should be enough to be able to start working with JIT compilation using MCJIT. More on that later.

 [pass_registry_initialisation]: http://llvm.org/docs/doxygen/html/group__LLVMCInitialization.html
 [rust_once]: https://doc.rust-lang.org/std/sync/struct.Once.html