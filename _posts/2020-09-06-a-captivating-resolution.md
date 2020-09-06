---
title: A Captivating Resolution
layout: post
published: true
---

Compilers are generally split into three stages: syntax, semantic, and emit. The translation from syntax tree, produced by the first stage, to semantic tree, consumed by the latter stages, is called *binding*. Binding involves walking the syntactic representation, resolving variable references, and producing a tree based on the semantic structure of the program. For a single scope variable lookup is fairly simple. When dealing with lambdas that can capture values from an outer scope things get more complex. In this post I'll walk through developing a simple binder that can handle lambda captures.

We will begin by assuming our input has already been parsed and is provided as a tree of `SynNodes`.

```fsharp
/// Syntactic node, as if from source code.
type SynNode =
    | Number of int
    | Define of string * SynNode
    | Load of string
    | Store of string * SynNode
    | Lambda of string * SynNode
    | Seq of SynNode list
```

Our bound tree will be made up of another union. We'll begin by just supporting numbers:

```fsharp
/// Bound node, with captured variables resolved
type Bound =
    | Number of int

/// Bind the syntactic tree and produce a bound tree
let rec private bind = function
    | SynNode.Number n -> Bound.Number n
    | e -> failwithf "Error binding %A" e
```

Numbers are simple to bind as no resolution needs to happen.

<table>
<thead><tr><th>Syntax</th><th>Bound</th></tr></thead>
<tbody>
<tr><td><pre>Number 123</pre></td><td><pre>Number 123</pre></td></tr>
</tbody>
</table>

Next up is variable definitions. In this language we assume a variable needs to be *defined* before being used. To support this we need some place to store the variables in the current scope. To keep track of the defined variables along with other state as we perform the bind we introduce a `BindCtx` type:

```fsharp
/// Locals are indices into some arbitrary array of
/// storage. Locals begin their indexing again at 0
/// when a new lambda is defined.
type Local = Local of int

/// Binder context, flowed through the bind to keep
/// track of the current state.
type BindCtx =
    { mutable Locals: string list }
    
    /// Introduce a new local definition.
    member ctx.Define id =
        let idx = ctx.Locals.Length
        ctx.Locals <- id::ctx.Locals
        Local idx
```

With that in place we can now bind definitions:

```fsharp
type Bound =
    | Store of Local * Bound
    // ...

let rec private bind (ctx: BindCtx) = function
    | SynNode.Define(id, init) ->
        let local = ctx.Define id
        Bound.Store(local, bind ctx init)
    // ...
```

Defining variables introduces a new local to the binder context, and then represents the initialisation in the bound tree as a plain `Store` operation.

<table>
<thead><tr><th>Syntax</th><th>Bound</th></tr></thead>
<tbody>
<tr><td><pre>Define ("foo",Number 123)</pre></td><td><pre>Store (Local 0,Number 123)</pre></td></tr>
<tr><td><pre>Define ("foo",Define ("bar",Number 456))</pre></td><td><pre>Store (Local 0,Store (Local 1,Number 456))</pre></td></tr>
</tbody>
</table>

To bind plain loads and stores we need to look up the ID in the current context, and emit a bound operation that reads or writes to that location. Looking up the ID is performed by searching for a local variable in the list of locals. Once we have the index we need to "flip" it to return the offset from the base of our stack of local variables:

```fsharp
type BindCtx =
    // ..

    /// Lookup the given `id` in the current binder
    /// context.
    member ctx.Lookup id =
        List.tryFindIndex (fun l -> l = id) ctx.Locals
        |> Option.map (fun idx ->
	        Local (ctx.Locals.Length - 1 - idx))
```

Now we can look locals up in the environment adding loads and stores to the bind method is fairly simple. We search for the `id` and raise an error if no definition is found for it:

```fsharp
type Bound =
    | Load of Local
    // ...

let rec private bind (ctx: BindCtx) = function
    | SynNode.Load id ->
        match ctx.Lookup id with
        | Some local -> Bound.Load local
        | None ->
          failwithf "Reference to undefined %s" id
    | SynNode.Store(id, expr) ->
        match ctx.Lookup id with
        | Some local -> Bound.Store(local, bind ctx expr)
        | None ->
          failwithf "Assignment to undefined local %s" id
    // ...
```

<table>
<thead><tr><th>Syntax</th><th>Bound</th></tr></thead>
<tbody>
<tr><td><pre>Seq [
 Define (
  "foo",
  Number 234);
 Load "foo"
]</pre></td><td><pre>Seq [
 Store (
  Local 0,
  Number 234);
 Load (Local 0)
]</pre></td></tr>
</tbody>
</table>

Next up it's time to start work on the lambdas. In this language lambdas have a single argument and a body. The first change is to expand our idea of a variable to include arguments as well as lambdas:

```fsharp
/// Storage locations for variables.
type Storage =
    /// Locals are indices into some arbitrary array
    /// of storage. Locals begin their indexing again
    /// at 0 when a new lambda is defined
    | Local of int
    /// Load from the function's argument
    | Arg
    
type Variable =
    { Name: string
    ; Storage: Storage }
```

Lambdas will introduce a new scope on the stack. We will model this as a brand new `BindCtx`. Each nested context contains a reference back to the parent context. Instead of just a list of `string`s we now use our new `Variable` type.

```fsharp
type BindCtx =
    { Parent: BindCtx option
    ; mutable NextLocal: int
    ; mutable Locals: Variable list }
    
    // ..

    /// Create a derived contex for binding lambdas
    static member WithParent parent =
        { Parent = Some(parent)
        ; NextLocal = 0
        ; Locals = [] }
```

As well as our existing `Define` that needs updating to add a `Variable` to our `Locals` we add a `DefineArg` to introduce a definition for the lambda's argument.

```fsharp
    /// Introduce a new local definition.
    member ctx.Define id =
        let storage = Storage.Local(ctx.NextLocal)
        ctx.Locals <- { Name = id
                      ; Storage = storage }::ctx.Locals
        ctx.NextLocal <- ctx.NextLocal + 1
        storage

    /// Define a lambda argument
    member ctx.DefineArg id =
        let storage = Storage.Arg
        ctx.Locals <- { Name = id
                      ; Storage = storage }::ctx.Locals
```

With all this in place binding non-capturing lambdas is fairly simple. A new context is created with the current context as its parent. We define its argument and then bind the body with that new context. 

```fsharp
let rec private bind (ctx: BindCtx) = function
    // ...
    | SynNode.Lambda(formal, body) ->
        let lambdaCtx = BindCtx.WithParent(ctx)
        lambdaCtx.DefineArg formal
        bind lambdaCtx body |> Bound.Lambda
```

So long as the lambda doesn't reference a variable outside its scope all should be good.

<table>
<thead><tr><th>Syntax</th><th>Bound</th></tr></thead>
<tbody>
<tr><td><pre>Lambda ("x",Load "x")</pre></td><td><pre>Lambda (Load Arg)</pre></td></tr>
</tbody>
</table>

To resolve variable references from parent scopes things get a little more complex. If a lambda references a variable from one of the enclosing scopes it is said to 'capture' or 'close over' that value.

To allow the nested lambda to share access of this captured value with other scopes we move the storage location from the function's locals into some 'environment' structure. This extends the lifetime of the variable beyond the scope of the lambda it was defined in so references to it in child lambdas can outlive the initial call. Now if anyone wants to modify the value there's a centralised location to use. In the original scope access to the variable goes directly to the environment. In nested scopes we keep track of how far up the call chain we need to walk to find the variable's environment using a chain of `Capture`s.

The first change to achieve this is to have variable lookup in a given context fall back to searching in the parent. We use a special `ParentLookup` method here to encompass both the lookup and capture of the variable.

```fsharp
    /// Lookup the given `id` in the current binder.
    member ctx.Lookup id =
        match ctx.LookupVar id with
        // If this was in our environment, then return
        // the variable's current storage location
        | Some(v) -> Some(v.Storage)
        | None -> ctx.ParentLookup id
```

The `ParentLookup` itself first checks to see if the current scope even has a parent. If a parent is found we delegate the actual lookup to a _third_ method `LookupAndCapture`. If we find a variable then we return a capture reference to it.

```fsharp
    /// Look up the id in the parent context, and
    /// return an `Capture` to it.
    member ctx.ParentLookup id =
        match ctx.Parent with
        | Some(parent) -> 
            match parent.LookupAndCapture id with
            | Some(storage) ->
                Some(Storage.Capture(storage))
            | None -> None
        | None -> None
```

The `LookupAndCapture` method is quite similar to the plain `Lookup`, but moves values to the `Environment` if they don't already live there.

```fsharp
    /// Lookup the given `id` in the current context, and
    /// move it to captured storage if it hasn't already
    /// been moved there.
    member ctx.LookupAndCapture id =
        match ctx.LookupVar id with
        | Some(v) -> 
            match v.Storage with
            | Environment(_) -> Some(v.Storage)
            | s -> 
                v.Storage <- Storage.Environment(s)
                Some(v.Storage)
        | None -> ctx.ParentLookup id
```

With all this in place it should now be possible to bind lambdas that reference variables in an outer scope. There's one last wrinkle to iron out however. When we come to lower this code we need to know _when_ to move values into the environment. To do this we need to know what values have been captured by a given lambda. We'll add a list of the known captures to the `BindCtx`, and add to it each time we perform a `ParentLookup`:

```fsharp
/// Binder context, flowed through the bind to keep track of the current state
type BindCtx =
    { // ..
    ; mutable Captures: Storage list }
    
    member ctx.ParentLookup id =
        match ctx.Parent with
        | Some(parent) -> 
            match parent.LookupAndCapture id with
            | Some(storage) ->
                ctx.Captures <- storage::ctx.Captures
                Some(Storage.Capture(storage))
            | None -> None
        | None -> None
```

Now all that remains is to add this to our `Lambda` type, and set it when we bind:


```fsharp
type Bound =
    // ...
    | Lambda of Storage list * Bound
    
    // .. 
        let body = bind lambdaCtx body 
        Bound.Lambda(lambdaCtx.Captures, body)
```

<table>
<thead><tr><th>Syntax</th><th>Bound</th></tr></thead>
<tbody>
<tr><td><pre>Seq [
 Define (
  "a",
  Number 1337
 );
 Lambda (
  "x",Load "a"
 )
]</pre></td><td><pre>Seq [
 Store (
  Local 0,Number 1337);
 Lambda (
  [Environment (Local 0)],
  Load (
   Capture (
    Environment (Local 0))))]</pre></td></tr>
</tbody>
</table>

And that's it. We can now bind syntax trees into a semantic model that tracks _when_ captures need to take place and provides enough information to lower to some form of executable code. To view the solution in full check out the [Source Code for this Article](https://github.com/iwillspeak/captivating). For a more in-depth and slightly more complex solution that inspired this check out the [Feersum compiler's `bind` method](https://github.com/iwillspeak/feersum/blob/main/src/Feersum/Bind.fs).

## Further Reading

 * [Binder parent lookup in Feersum](https://github.com/iwillspeak/feersum/blob/1f57cbc1c5cf67cee29e63d40f86f1045e1dd9f0/src/Feersum/Bind.fs#L78-L102)
 * [Feersum Scheme - Captures part 1](https://www.youtube.com/watch?v=fzCeGEqjz-I&t=17s) - The stream that prompted this article.
 * [Closure Conversion in C#](https://github.com/dotnet/roslyn/blob/3115ff84ef7ea8c3b1989172605c51a18d04ba95/docs/compilers/Design/Closure%20Conversion.md) - the Roslyn compiler's approach to this problem.
 * [Closures - Crafting Interpreters](https://craftinginterpreters.com/closures.html) - a similar approach to captures and inspired by Lua's _upvalues_.
