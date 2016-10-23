---
title: The Low Down - Using LLVM From Rust
layout: post
published: true
---

LLVM is a collection of libraries and programs for creating compilers. It is built around a powerful intermediate representation, LLVM IR. It has tools for compiling LLVM IR to native instructions, performing optimisations and even performing JIT compilation. The libraries that power this have an [extensive C API][llvm_c] allowing you to harness this power yourself, so that’s what we’re going to do! I’ll show you how to create a simple Rust program which creates a simple function from scratch in LLVM.

First let’s get up and running. Start by creating a new Rust project adding a reference to the [`llvm-sys`][llvm_sys] crate to the `Cargo.toml` file:

    [dependencies]
    llvm-sys = "*"

With this in place it’s probably worth making sure everything will compile first. The _llvm-sys_ crate doesn’t contain a copy of LLVM itself, instead it relies on an existing installation. This should just be a matter of consulting your platform’s package manager. If you’re on Linux make sure to install the `llvm-dev` package. OS X users can find the `llvm` package in Homebrew. Because OS X comes with a version of LLVM itself Homebrew won’t add it to your path. To compile using the version Homebrew has installed you’ll need to modify your path first: `$ export PATH=/usr/local/opt/llvm/bin:${PATH}`. You can check you’ve got LLVM installed correctly by running `$ llvm-config —version` from the command line. You should see some output similar to the following:

    $ llvm-config --version
    3.8.1

If `llvm-config` can’t be found make sure that you’ve got a dev version of LLVM installed, along with the headers. On linux platforms these are usually suffixed with `-dev`. If that is the case make sure that your path includes the `/bin` folder from that distribution of LLVM.

If all is going well so far building your first Rust program which references LLVM should be just a `cargo build` away. If that all compiled cleanly then give yourself a pat on the back. Getting a rust program to build and link to LLVM is half of the battle.

Once the foundations are in place the next step is to start _using_ LLVM for something. All code in LLVM lives in a **Module**. Modules are created with a call to `LLVMModuleCreateWithName`. Because this is C land you’ll also have to make sure to clean up modules once you’re done with them by calling `LLVMDisposeModule`:

{% highlight rust %}

extern crate llvm_sys;

use std::ffi::CStr;

use llvm_sys::prelude::*;
use llvm_sys::{core, target};

fn main() {
    // Create a module to compile the expression into
    let mod_name = CStr::from_bytes_with_nul(b"hello\0").unwrap();
    let module = unsafe { core::LLVMModuleCreateWithName(mod_name.as_ptr()) };
    
    // ...
    
    // Clean up the module after we're done with it.
    unsafe { core::LLVMDisposeModule(module) }
}
{% endhighlight %}

Although this doesn’t do anything incredibly interesting at the moment it’s worth noticing a few things. Firstly we are importing a few things from `llvm_sys`. The create is organised into a series of Rust modules named after the header file from the C api. The `prelude` contains most of the pointer & type definitions that are shared. Secondly seen as we’re using the `llvm-sys` crate here we’re dealing with raw bindings to C functions. That means that we need to pepper our code with `unsafe` to call any of them. It is general good practice in production Rust code to wrap these unsafe pointers and functions up in ‘safe’ Rust wrappers to keep the `unsafe` separate from the main logic of your program, and make sure that objects are free’d properly using the [`Drop` trait][std_drop]. I’m not doing that here though as the aim of this post is to demonstrate the use of the API itself.

Next up it’s time for some Types. LLVM IR is similar in some ways to assembly code but it contains far more type information. Before we can create a function in our module we need to create a type for it. Basic types already exist for things like integers. For more complex types such as function types we need to call a function to ‘build’ it from its constituent parts:

{% highlight rust %}
 // Create a function type and add a function of that type to the module
let function_type = unsafe {
    let int64 = core::LLVMInt64Type();
    let mut param_types = [int64, int64];
    core::LLVMFunctionType(int64, param_types.as_mut_ptr(), param_types.len() as u32, 0)
};
{% endhighlight %}

Once we have the type for our function adding it to the module is simple. We just need to give it a name:

{% highlight rust %}
let function_name = CStr::from_bytes_with_nul(b"test\0").unwrap();
let function =
    unsafe { core::LLVMAddFunction(module, function_name.as_ptr(), function_type) };
{% endhighlight %}  

You can’t go straight ahead and start adding code to this function. LLVM IR is built up from a ‘graph’ of _basic blocks_. Each block contains code which is executed in series and ends with either a return statement or a jump to another block. Our function will be super simple so we just need a single block. One thing to note here is that we give it a name. This acts as a label and allows control flow to jump to the start of the block.

{% highlight rust %}
// Create a basic block and add it to the function
let block_name = CStr::from_bytes_with_nul(b"entry\0").unwrap();
let entry_block = unsafe { core::LLVMAppendBasicBlock(function, block_name.as_ptr()) };
{% endhighlight %}

The easiest way to instructions into this block is using an IR Builder. This provides a cursor into a basic block where instruction will be inserted and [a large number of methods for creating new instructions][ir_builder]. You only need one of these objects. Once you’re done inserting instructions into one basic block you can point it at another with `LLVMPositionBuilderAtEnd` and use it again.

{% highlight rust %}
// Create an LLVM instruction builder and position it at our new
// basic block
let builder = unsafe { 
    let b = core::LLVMCreateBuilder();
    core::LLVMPositionBuilderAtEnd(b, entry_block);
    b
};
{% endhighlight %}

Now we have a builder it’s time to create the body of our function. First we get references to each of the function’s parameters using `LLVMGetParam`. We then insert an add instruction to sum up both of these values. Like most instructions in LLVM this creates a new value, and we give that value a name. Don’t worry about the number of variables this creates. When optimisations are run most of these ‘intermediate values’ will be removed.

{% highlight rust %}
// Fill in the body of the function.
unsafe {
    let a = core::LLVMGetParam(function, 0);
    let b = core::LLVMGetParam(function, 1);

    let temp_name = CStr::from_bytes_with_nul(b"temp.1\0").unwrap();
    let res = core::LLVMBuildAdd(builder, a, b, temp_name.as_ptr());

    core::LLVMBuildRet(builder, res);
}
{% endhighlight %}

And that’s it. You should now have a representation of a simple function in memory which adds together two 64 bit numbers. All that’s left is to clean up our builder object:

{% highlight rust %}
// Clean up the builder now that we are finished using it.
unsafe { core::LLVMDisposeBuilder(builder) }
{% endhighlight %}

From here you could compile this module down to native code or JIT compile it and run it in the same process. Maybe I’ll cover that in a later post. Right now let’s dump the whole module to see what LLVM has created for us:

{% highlight rust %}
// Dump the LLVM IR to stdout so we can see what we've created
unsafe { core::LLVMDumpModule(module) }
{% endhighlight %}

Compile and run and you should see a textual representation of the LLVM IR we built:

{% highlight llvm %}
; ModuleID = 'hello'

define i64 @test(i64, i64) {
entry:
  %temp.1 = add i64 %0, %1
  ret i64 %temp.1
}
{% endhighlight %}

I hope you’ve enjoyed yourself. Who knows maybe this even proves useful to someone. If it does please do let me know in the comments. If i’ve made a mistake somewhere, or mis-represented the LLVM API then please let me know. I’m only just getting started with it myself.

You can find the full code for this article [in this gist][gist]. You should be able to clone the gist directly and compile it with Cargo.

## Further Reading

 * The [LLVM Sys examples][llvm_sys_examples] provide a lot of useful tidbits.
 * Alex Denisov has a useful article about [getting started with LLVM from a Swift point of view][swift_llvm].
 * Wilfred Hughes’ [optimising Brainfuck compiler][bfc] is a great use of the `llvm-sys` crate.

[gist]: https://gist.github.com/iwillspeak/1fed07333f7187c25735e90d73b82468#file-main-rs
[swift_llvm]: http://lowlevelbits.org/how-to-use-llvm-api-with-swift/
[llvm_sys]: https://docs.rs/crate/llvm-sys/0.4.0	
[llvm_c]: http://llvm.org/docs/doxygen/html/group__LLVMC.html
[llvm_sys_examples]: https://bitbucket.org/tari/llvm-sys.rs/src/default/examples/	
[ir_builder]: http://llvm.org/docs/doxygen/html/group__LLVMCCoreInstructionBuilder.html
[bfc]: https://github.com/Wilfred/bfc
[std_drop]: https://doc.rust-lang.org/std/ops/trait.Drop.html