---
title: Rusting up Some Iron
layout: post
published: true
---

Rust is one of my favourite languages. One of it's great strengths is its ecosystem of *crates*, another is the FFI system which enables writing native extensions when performance or safety is required. Over the last week I've been pulling these two together in [IronRure](https://github.com/iwillspeak/Ironrure), a .NET Standard wrapper around the Rust `regex` crate.

I started this with zero knowledge of FFI in C#. I was pleasantly surprised with how easy P/Invoke was to use. There's plenty of documentation on its use from both [Microsoft](https://docs.microsoft.com/en-us/dotnet/articles/standard/native-interop) and [Mono](http://www.mono-project.com/docs/advanced/pinvoke/). I feel I should pull together the lessons I've learned though. Don't hesitate to comment if you think I'm *Doing it Wrong*.

## Stage 1 - Quick and Dirty

The first step is to get the compiled Rust code next to your managed assembly so that P/Invoke can find it and load it. The simplest way to get this working is to compile the Rust ahead of time with `cargo` and add a direct `Content` reference  to the output. Here's what I started with:

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>netstandard1.4</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <Content Include="..\..\thirdparty\regex\regex-capi\target\release\rure.dll">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </Content>
  </ItemGroup>
</Project>
```

Since I was targeting .NET Standard I wanted this to work on Windows, Mac and Linux. To solve that we'll need to wrap the native code in a NuGet package. Content files will get you a long way though when messing about with an idea.

## Going Cross Platform

The first step in making sure that your bindings will work on different platforms is to make sure that `DllImport` statements aren't platform specific. Luckily Rust libraries have a well known naming structure which matches up with what .NET expects. 

```csharp
[DllImport("foo")]
public static extern void hello();
```

By using `DllImport` wihout a `lib` prefix and `.dll`/`.so`/`.dylib` suffix the runtime will pick the correct file on Linux, Windows and Mac!

## Packing it Up

Although .NET code is not targeted to a specific CPU your native Rust code is. In .NET the underlying target architecture is called the [Runtime Identifier](https://docs.microsoft.com/en-us/dotnet/articles/core/rid-catalog), or RID for short. RIDs are arranged into a tree. If you're running on `win10-x64` the runtime will try to load native code for the `win7-x64`, `win-x64` and `win` RIDs when installing a NuGet package. We can exploit this to create a set of native-only NuGet packages containing just the compiled Rust code which are then referenced as NuGet dependencies in the managed FFI wrapper.

You could create a different NuGet package for each runtime identifier being targeted. I chose however to create a single package for each operating system and include all the runtime identifiers I was supporting for that OS. On Mac there is only a single RID to target really: `osx`. After compiling the crate (using the `--release` flag) the output needs to be copied so it has the following structure:

```
contents/
└── runtimes
    └── osx
        └── native
            ├── librure.a
            ├── librure.d
            └── librure.dylib
```

We can then pack this up into a NuGet package with the following `.nuspec` file:

```xml
<?xml version="1.0" encoding="utf-8"?>
<package>
  <metadata>
	  ...
  </metadata>
  <files>
    <file src="contents\**" />
  </files>
</package>
```

The FFI wrapper library just needs to add a `PackageReference` like any other library. When you compile the appropriate native library will be picked from its NuGet and placed next to the output program ready for P/Invoke to find.