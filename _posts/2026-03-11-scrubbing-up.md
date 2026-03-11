---
group: blog
title: Scrubbing Up
layout: post
published: true
---

One of the key features that makes Scheme such a powerful language is the support for powerful macro transformations. A Scheme program goes through a stage of syntax expansion before the program is bound. Many syntax constructs in Scheme programs are in fact macros. So far in Feersum our macro support has been lacking a core concept however: hygiene. In this post I aim to outline the shape of the new macro system and how it will provide support for hygienic macro expansion.


In their [1988 paper "Syntactic Closures"](https://dspace.mit.edu/handle/1721.1/6036) Allan Bawden and Jonathan Rees wrote:

> One way to avoid capture problems (like `or`'s problem with `temp`) is to use names so obscure that the `macro`'s client is unlikely to discover them accidentally.

If we inspect the [current source code of Feersum](https://github.com/iwillspeak/feersum/blob/6e67422399b7892c5c757e6130c858e7a6cf45fb/src/Feersum.CompilerServices/Compile/Builtins.fs#L139-L145) we see that we are in fact being called out by a pair of computer scientists from the 80s:

```scheme
(syntax-rules ()
  ((or) #f)
  ((or test) test)
  ((or test1 test2 ...)
   (let ((|90a3b246-0d7b-4f47-8e1e-0a9f0e7e3288| test1))
     (if |90a3b246-0d7b-4f47-8e1e-0a9f0e7e3288| |90a3b246-0d7b-4f47-8e1e-0a9f0e7e3288| (or test2 ...)))))
```

The only real response to such a cutting criticism is to _actually do it properly_. In the rest of this post I'll try and summarise the general approach discussed by Bawden and Rees and sketch an implementation of Syntactic Closures for a toy language.


## Environmental Issues

When we evaluate a program an interesting interaction takes place between the `names` in our source text and the `variables` that they refer to. It's so natural that it often goes unnoticed. Let's refresh ourselves just the same. Say we have a simple lambda calculus program expressed in s-expression form:


```lisp
((lambda x x) 10)
```

In such a program we can trivially see the result should evaluate to `10`. The way this works can be modelled out through maths as a series of reduction rules. Seen as we are talking about _programming_ here however the other option is to "evaluate" this expression. To do this we recurse into each nested form carrying an __execution environment__ with us. This environment is a mapping from the names in our program to the values we store for them.

In our trivial program there is only one name, `x`, and only one location that this refers to; the argument to our lambda. In a more complex program however the name `x` may refer to different locations depending on the __execution scope__ within the program. Take for example this updated program which triples a value:


```lisp
((lambda x (+ x
            ((lambda x (+ x x)) x))) 10)
```

Here in the outer lambda `x` is _bound to_ the outer lambda's argument. In the inner lambda it is instead bound to the inner lambda's argument. This is a concept commonly known as "shadowing".

A subtle variation introduces the key third concept we need to understand: closures. If instead of using the same name `x` to refer to both parameters we instead named the inner one `y` the outer `x` is no longer shadowed. This allows us to capture, or __close over__ the outer `x` in the body of the inner lambda.

```lisp
(((lambda x (lambda y (+ x y))) 100) 50)
```

In this program the inner lambda "remembers" information about the execution environment in which it was created. In the Syntactic Closures paper an orthogonal approach is laid out that allows __syntax__ to remember the _syntactic environment_ in which it was created to prevent names accidentally shadowing values in their new locations within the expanded program once syntax transformation is performed.


## The Syntax Domain

Just as programs being executed have an execution environment and runtime closures which are used to bind meanings and context to names the same can be done in the syntax domain. This is the core abstraction proposed in the Syntactic Closures paper.

Let's imagine our language has another special form as well as `lambda` called `def-syntax`. This form is responsible for defining a simple _syntax transformer_, or "macro". Our macros will take a list of arguments and expand into some arbitrary syntax. The two sides of each rule within our syntax transformer are the _pattern arguments_ on the left, and the _template_ on the right:

```lisp
(def-syntax or
	((_ a) a)
	((_ a b) ((lambda x (if x x b)) a)))
```

Here our definition of `or` has two cases. For a single value we simply expand to that value. For two values we expand to a lambda which binds the first value to the argument `x`. This then allows us to check and return `x` without evaluating it twice.

When our compiler encounters this special form at syntax **expansion** time, which takes place before the evaluation of the program, we perform some actions which feel familiar. First we convert the body of the `def-syntax` into a transformer and then we record in our _syntax environment_ that the name `or` now refers to that macro. Within the remaining scope of the program future references to `or` will be treated as applying this transformer.

When we see a reference to the macro, such as `(or 100 200)` we first check each rule in turn to find a match based on the pattern. Once we have found a match we bind the values `a -> 100` and `b -> 200` in the syntax environment and move on to expand the template. From now on references in the template to the names `a` or `b` will be replaced, or _substituted_, with their respective values from the syntax environment.

This substitution on its own however is not sufficient to provide hygienic expansion. To do this we need to introduce two new concepts. The first is the idea of an `identifier`. An `identifier` is a pairing of a name and an opaque "stamp". In our examples we'll use an incrementing integer. These `identifier`s allow us to tell the various different `x`s apart in our syntax. Thinking back to our nested example from earlier if we annotate it with the identifiers in place of the names it might become:

```lisp
((lambda x.1 (+ x.1
            ((lambda x.2 (+ x.2 x.2)) x.1))) 10)
```

Here we can see that the "outer `x`" and "inner `x`" are two logically different identifiers. This transformation is analogous to alpha renaming.

In addition to binding `name`s to `identifier`s we can also introduce the idea of a **Syntactic Closure**. The idea here is that when a piece of syntax "escapes" from its place within the static structure of a program and identified scoping we need to capture the state of the syntax environment at that point so that it can be restored later; just as calling a lambda allows it to access the "remembered" context that it captured in the execution environment.

A syntactic closure contains two things:

 * A syntax environment. This is a mapping from `names` to their "syntactic item". This could be the fact that `x` refers to the identifier `x.1` or it could be that `or` refers to the transformer for our `or` macro.
 * A piece of syntax to apply it to.

With these two pieces in place we can now sketch out how a language can perform hygienic expansion of the `or` macro using Syntactic Closures.

## The `Cst` Tree

We begin from a simple tree node type which directly encodes the concrete syntax of a simple language:

```fsharp
type Cst =
  | Symbol of string
  | Literal of int
  | Form of Cst list
```

This encodes the raw syntactic structure of our language. This will not be sufficient to model our understanding of the syntax tree outlined above however. To solve this we introduce a second parallel `stx` tree. Once our parser has produced the `Cst` we use the `illum` function to walk the concrete syntax and produce a new tree where each raw symbol has been given its initial binding to an identifier. First our tree structure.

```fsharp
type Ident = { Name: string; Stamp: int }

type Stx = 
  | Literal of int
  | Ident of Ident
  | Form of Stx list
  | Closure of Stx * StxEnv

and StxEnv = Map<string, StxBinding>

and StxBinding =
  | Lam
  | Var of Ident
  | Macro of Transformer

and Transformer = Stx -> StxEnv -> Stx
```

The `illum` function then takes our `Cst` and maps all names to identifiers with the stamp `0`:

```fsharp
let public illum = function
  | Cst.Symbol s ->
    Stx.Ident({ Name: s; Stamp: 0})
  | Cst.Literal l -> Stx.Literal l
  | Cst.Form f ->
    f
    |> List.map (illum)
    |> Stx.Form
```

We then define a pair of functions to help us deal with syntax environments. The first takes a piece of syntax and wraps it into a `Stx.Closure`. The second resolves a given name _in the `StxEnv`_ and returns the `StxBinding`. If no entry exists in our syntax environment for a name we return the 'default' meaning for that name. For our special form `lambda` this is our token `Lam` binding, for all other names we represent them as "free variables".

```fsharp
let close (stx: Stx) (defEnv: StxEnv) : Stx =
  StxClosure(stx, defEnv)

let resolve (id: Ident) (stxEnv: StxEnv) : StxBinding =
  match stxEnv.TryFind id.Name with
  | Some binding -> binding
  | None ->
    match id.Name with
    | "lam" -> Lam
    | _ -> Var id
```

With these two functions in place we can begin to implement our `expand` method. It takes a piece of syntax and a syntax environment and produces an optional replacement piece of syntax and a new environment. The first output is `option` as some syntax items, such as `def-syntax` will have no representation in the final tree. In this case we would return `None` and elide it from the final expanded form of the program. The second output represents the new, potentially expanded, environment in which further syntax should be expanded. Using these two items we can then perform a `mapFold` over a sequence of `Stx` items in order to expand a program.

```fsharp
let rec public expand (stx: Stx) (stxEnv: StxEnv) : Stx option * StxEnv =
```

For the first few cases our syntax expansion is relatively trivial. Empty forms expand to themselves, and so do literals:


```fsharp
  match stx with
  | Stx.Literal _
  | Stx.Form [] -> (Some stx, stxEnv)
```

For `identifier`s we need to resolve them to their binding in the syntax environment. If they are not bound to a variable this is an error. References to syntax items should only be at the head of a form in order to apply them.

```fsharp
  | Stx.Ident id ->
    match (resolve id stxEnv) with
    | Var id -> (Some Stx.Ident(id), stxEnv)
    | _ -> failwith "Unexpected syntax item in value position"
```

The interesting cases arise when we encounter a non-empty form. The head of the form determines what kind of expansion needs to take place. If the head is an identifier we resolve it in the syntax environment, just as before, and then dispatch on the result:

```fsharp
  | StxForm (head :: args, f) ->
    match head with
    | StxIdent(id, sym) ->
      let resolved = resolve id stxEnv
      match resolved with
      | Macro transformer ->
        (Some (transformer stx stxEnv), stxEnv)
      | Lam -> (Some (expandLam head args f stxEnv), stxEnv)
      | DefSyn -> expandDefSyn args stxEnv
      | Var _ ->
        let expanded =
          StxForm(List.map (expandOne stxEnv) (head :: args), f)
        (Some expanded, stxEnv)
    | _ ->
      let expanded =
        StxForm(List.map (expandOne stxEnv) (head :: args), f)
      (Some expanded, stxEnv)
```

There are four cases for a resolved head identifier. If it's a `Macro` we invoke the transformer directly — we'll see how transformers work shortly. If it's the special form `Lam` we delegate to `expandLam`. If it's a `DefSyn` we delegate to `expandDefSyn`, which returns `None` and an updated environment rather than a piece of expanded syntax. For anything else — an ordinary variable reference in head position, or a non-identifier head — we simply recurse into the sub-forms.

The `expandOne` helper is a thin wrapper that enforces that we only call `expand` in positions where a result is required:

```fsharp
and private expandOne (stxEnv: StxEnv) (stx: Stx) : Stx =
  match expand stx stxEnv with
  | Some result, _ -> result
  | None, _ -> failwith "def-syn not valid in expression position"
```

## Renaming Binders

When expanding a `lam` form we encounter the core of the hygiene mechanism. The parameter identifier needs to be given a fresh stamp so that it cannot accidentally shadow or be shadowed by identifiers with the same name introduced elsewhere — either by the user or by a macro.

```fsharp
and expandLam head args low stxEnv =
  match args with
  | StxIdent(id, s) :: body ->
    let (innerEnv, renamedId) = rename stxEnv id
    let expandedBody = List.map (expandOne innerEnv) body
    StxForm(head :: StxIdent(renamedId, s) :: expandedBody, low)
  | _ -> failwith "Invalid syntax for lam: expected (lam <id> <body>)"
```

The `rename` function produces a fresh identifier with a new stamp for the binding site and adds a `Var` mapping to the syntax environment so that any reference to the same _name_ within the body will resolve to that fresh identifier. This is the key to preventing accidental capture: once a binder is renamed, all references within its scope pick up the renamed identifier through `resolve`. References originating from _outside_ the lambda body — for example, free variables in a macro template — will carry a different stamp from when they were illuminated or closed over, and so will resolve differently.

Thinking back to our nested lambda example from earlier, `expandLam` is what ensures the two `x` binders end up as `x.1` and `x.2` rather than competing for the same name.

## Defining Syntax

The `expandDefSyn` function handles the `def-syntax` special form. It parses each rule, builds a transformer, and returns an updated syntax environment:

```fsharp
and private expandDefSyn
    (args: Stx list) (stxEnv: StxEnv) : Stx option * StxEnv =
  match args with
  | StxIdent(macroName, _) :: ruleStxs ->
    let rules = List.map parseRule ruleStxs
    let transformer = makeSynTransformer rules stxEnv macroName.Name
    let newEnv = Map.add macroName.Name (Macro transformer) stxEnv
    (None, newEnv)
  | _ -> failwith "Invalid def-syn form: expected (def-syn <name> <rule>...)"
```

Note that `expandDefSyn` returns `None` for the syntax item itself — a macro definition produces no output in the expanded tree, only a change to the environment.

Parsing a rule simply extracts the pattern argument identifiers and the template from the rule's structure:

```fsharp
and private parseRule (ruleStx: Stx) : MacroRule =
  match ruleStx with
  | StxForm([StxForm(StxIdent _ :: patArgs, _); template], _) ->
    let idents = patArgs |> List.mapi (fun i pat ->
      match pat with
      | StxIdent(id, _) -> id
      | _ ->
        failwith (
          $"Invalid pattern in macro rule at position {i}: " +
          $"expected an identifier, got %A{pat}"))
    { PatArgs = idents; Template = template }
  | _ ->
    failwith (
      $"Invalid macro rule: expected ((name pat...) template), " +
      $"got %A{ruleStx}")
```

## The Transformer and the Syntactic Closure

The real payoff arrives in `makeSynTransformer`. This is where the definition-time environment is captured — forming the syntactic closure — and where it is later used to wrap the expanded template:

```fsharp
and private makeSynTransformer
    (rules: MacroRule list) (defEnv: StxEnv)
    (macroName: string) : Transformer =
  fun (callStx: Stx) (useEnv: StxEnv) ->
    match callStx with
    | StxForm(_ :: callArgs, _) ->
      let tryRule (rule: MacroRule) =
        match matchPatternArgs rule callArgs useEnv with
        | Some bindings ->
          let substituted = applyTemplate rule.Template bindings
          Some (StxClosure(substituted, defEnv))
        | None -> None
      match List.tryPick tryRule rules with
      | Some expanded ->
        match expand expanded useEnv with
        | Some result, _ -> result
        | None, _ ->
          failwith $"macro '{macroName}': def-syn in template is not valid"
      | None -> failwith $"No matching rule for macro '{macroName}'"
    | _ -> failwith $"Invalid macro call form for '{macroName}'"
```

The transformer closes over `defEnv` — the syntax environment at the point the macro was _defined_. When the transformer is later _invoked_, it receives `useEnv` — the environment at the call site. Pattern matching happens against `useEnv`, so the arguments from the call site are bound correctly. The template however is wrapped in a `StxClosure` carrying `defEnv`. When `expand` later encounters this closure it will expand the template's contents in the definition-time environment, not the call-site environment.

This is the final case in `expand` — the one we have been building towards:

```fsharp
  | StxClosure(stx, env) ->
    let (result, _) = expand stx env
    (result, stxEnv)
```

When we hit a syntactic closure we expand its inner syntax using the _captured_ environment, then return the result into the _outer_ environment unchanged. The two environments never mix. An identifier `x` introduced by the macro template can only ever resolve to what `x` meant where the macro was written — not to whatever `x` might mean at the call site. The Bawden and Rees paper would approve.

Returning to our motivating example, when `(or 100 200)` is expanded the template `((lambda x (if x x b)) a)` is wrapped in a closure over the definition-time environment. When that closure is later expanded, the `x` in the template resolves to a freshly stamped identifier created during `expandLam` — one that is entirely distinct from any `x` the caller might have in scope. No GUIDs required.

## Much Ado About Mutton

For the full code of this toy example, including handling for `def`initions as well as a `bind` pass which assigns storage locations to items you can check out the full [iwillspeak/Mutton](https://github.com/iwillspeak/Mutton) on GitHub. For the final implementation of hygiene in the Feersum compiler see the (currently work-in-progress) [Hygenic Macro Expansion PR](https://github.com/iwillspeak/feersum/pull/74).
