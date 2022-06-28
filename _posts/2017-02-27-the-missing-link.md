---
title: The Missing Link
layout: post
published: true
---

LLVM is great for hacking about with languages. It takes a lot of the complexity out of code generation. It allows you to turn your `SyntaxTree` or whatever you've got into real live object code. Code you can directly execute. Well, almost.

The last crevasse that LLVM plonks us down next to is **linking**. Taking a collection of object files, adding the C runtime library and producing an executable isn't *quite* as simple as it first seems. Luckily however the's a cunning trick to sidestep the problem: let someone else do it for you. The wonderful developers who make Clang have already solved those issues. Suddenly compiling down to an independent executable seems a lot easier.

I'm going to dive straight in and assume there's an `LLVMModuleRef` compiled and ready to go. The first thing we need to do is write the module out somewhere to disk where `clang` can read it from. In Rust I'm reaching for the [`tempdir` crate](https://crates.io/crates/tempdir) to make sure things get cleaned up afterwards, but really the exact place these intermediate files get written doesn't matter.

```rust
// Create a tempdir to write the LLVM IR to
let tmp_dir = TempDir::new("output")
    .expect("create temp dir");
let temp_path = tmp_dir.path().join("temp.ll");
```

Writing out an object file requires using the LLVM TargetMachine infrastructure. That sounds a bit fiddly though so it's easier to just write out LLVM IR for now with `LLVMPrintModuleToFile`.

```rust
let path = CString::new(temp_path.to_str().unwrap()).unwrap();
unsafe {
    let mut message = ptr::null_mut();
    if core::LLVMPrintModuleToFile(self.raw, path.as_ptr(), &mut message) == 0 {
        Ok(())
    } else {
        let err_str = CStr::from_ptr(message)
            .to_owned()
            .into_string()
            .unwrap();
        core::LLVMDisposeMessage(message);
        Err(err_str)
    }
}
```

With that in place all we need to produce a fully functioning executable is to shell out to Clang! In rust this is super easy with the `Command` builder. We just need to tell Clang where our LLVM IR is, and where to put the result when it's done.

```rust
// Shell out to Clang to link the final assembly
let status = try!(Command::new("clang")
    .arg(temp_path)
    .arg("-o")
    .arg("a.out") // It's traditional :-p
    .status());
```

And, if all goes well, Clang should do our dirty work and leave us with an executable `a.out`. All linked and ready to go!

I'm using this trick [in anger at the moment](https://github.com/iwillspeak/ullage/blob/02bae0bd428b49a6ea2f044796384be18c3179c7/src/compile/mod.rs#L82) as I continue hacking about on [Ullage](http://github.com/iwillspeak/ullage). I've since seen that Wilfred [has done the same thing for bfc](https://github.com/Wilfred/bfc/blob/11396b95647ecfc7f7e4e9e41e03bc692177baed/src/main.rs#L225).