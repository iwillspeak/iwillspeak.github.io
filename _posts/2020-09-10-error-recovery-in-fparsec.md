---
title: Error Recovery in FParsec
layout: post
published: true
---

An interesting discussion on [error recovery with the `nom` parser combinator in Rust](https://www.eyalkalderon.com/nom-error-recovery/) has been sitting on my "To Read" list for a few months. After re-reading it a few times I have been inspired to try out the approach in that article using the [_FParsec_ library][fparsec].

Just as in that example we will attempt to parse the language defined by the following _Parser Expression Grammar_ (PEG):

```peg
expr    ← sum
sum     ← product (('+' / '-') product)*
product ← value (('*' / '/') value)*
value   ← [0-9]+ / '(' expr ')'
```

The grammar is simple enough, and translating it into a parser combinator is fairly straightforward. Each non-terminal, on the left of the `←`, becomes a parser function. Each production on the right is mapped into the appropriate FParsec primitives:

```fsharp
let expr, exprRef = createParserForwardedToRef()

let value =
    (pint64  |>> Ast.Value ) <|>
    between (pchar '(') (pchar ')') expr

let product =
    let op = pchar '*' <|> pchar '/'
    (value .>>. (many (op >>. value)))
    |>> binNode Product

let sum =
    let op = pchar '+' <|> pchar '-'
    (product .>>. (many (op >>. product)))
    |>> binNode Sum

exprRef := sum
```

So far so good. Parsing valid inputs produces a `ParseResult.Ok` containing our syntax tree and all is well.

```
'123' ~> Value 123L
'1*2' ~> Product (Value 1L,Value 2L)
'(1+2)*3' ~> Product (Sum (Value 1L,Value 2L),Value 3L)
```

The problem arises when the parser fails.

```
Error in Ln: 1 Col: 2
(*1)+2)
 ^
Expecting: integer number (64-bit, signed) or '('
```

FParsec has done an excellent job of making an error message for us. This is a good example of how a parser combinator library can make knocking together a quick parser a breeze. There are a few issues however:

 1. It would be _super nice_ if we always got _some form_ of parse tree from our parser.
 2. Our example `(*1)+2)` has more than one syntax error in it.

 Parsers are generally used in two places. Either as part of a batch program performing a one-off compilation, or by a language server or IDE performing interactive syntax analysis. In either case, we want to provide as much information back to the programmer as possible. If we can get our hands on _some form_ of syntax tree we can still provide a gracefully degraded IDE experience. Recognising all the syntax errors we can at once is helpful to the programmer who is in a tight edit -> compile loop.
 
 The plan to solve this is simple.
 
 > The parser should __always__ succeed.

Instead of the current output of the parser, which is just a `SyntaxNode`, we instead would like it to return a pair of `SyntaxNode * Diagnostic list` where `Diagnostic` is some structure containing a `Position` and an error message.

To achieve this we will take advantage of FParsec's `UserState` to build our diagnostic list, and we will create some new parser combinators to backtrack on failure and emit diagnostics where needed.

First lets create a `State` type to be used by our parser:

```fsharp
type Diagnostic = Diagnostic of Position * string

type State =
    { mutable Diagnostics: Diagnostic list }
```

Simple enough, we have a mutable list of diagnostics. Let's add a member function to encapsulate emitting a diagnostic:

```fsharp
    member s.EmitDiagnostic pos err =
        let diag = Diagnostic(pos, err)
        s.Diagnostics <- diag::s.Diagnostics
```

Nice. Now let's create some parser combinators that use them. First we introduce the `expect` combinator. It takes a parser, and an error message. If the parser succeeds we wrap its return in a `Some`. If it fails we emit a diagnostic and return `None`.

```fsharp
let expect (p: Parser<'a, State>) e =
    let raiseErr (stream: CharStream<State>) =
        stream.UserState.EmitDiagnostic stream.Position e
        Reply(None)
    attempt p |>> Some <|> raiseErr
```

Our next is `expectSyn`. This is a thin wrapper over `expect` which maps failure to a new `SyntaxNode.Error` kind.

```fsharp
let expectSyn (p: Parser<SyntaxNode, State>) err =
    expect p err |>> Option.defaultValue SyntaxNode.Error
```

With this in place we can now start to re-write our parser to emit diagnostics and continue in the case of some errors. The general approach here is to wrap an `expect` over any right hand parts of a production that _should not_ fail.

```fsharp
let sum =
    let op = pchar '+' <|> pchar '-'
    let err = "expected expression after operator"
    (product .>>.
        (many (op >>. expectSyn product err)))
    |>> (binNode Sum)
``` 

In our `sum` parser we don't wrap the `op`, because we expect that to fail when there _isn't_ a `+` or `-`. If there _is_ an operator however we don't expect the right hand side to fail, so we use `expectSyn` to ensure it doesn't. A similar transform is needed for the `product` parser. With this in place our parser succeeds even on invalid inputs:

```
'1+' ~>
  ([Sum (Value 1L,Error)],
   [Diagnostic (
     ("test", Ln: 1, Col: 3),
     "expected expression after operator")])
```

Here the right-hand side of our `Sum` expression has been stubbed out with an `Error`. We have a diagnostic too, with a location pointing to where the problem lies.

Buoyed on by this success, let's add some `expect`s to the `value` parser:

```fsharp
let value =
    let opn = pchar '('
    let close = expect (pchar ')') "missing closing ')'"
    let body =
      expectSyn expr "expected expression after ("

    (pint64  |>> Value ) <|> between opn close body
```

We use a plain `expect` for the closing `)`, and `expectSyn` for the body of the parenthesised expression. Once we have seen an opening `(` neither of these are optional.

Now we can recognise errors in even more places:

```
'(1' ~>
  ([Value 1L],
   [Diagnostic (
     ("test", Ln: 1, Col: 3),
     "missing closing ')'")])
'()' ~>
  ([Error],
   [Diagnostic (
     ("test", Ln: 1, Col: 2),
     "expected expression after (")])
'(' ~>
  ([Error],
   [Diagnostic (
     ("test", Ln: 1, Col: 2),
      "missing closing ')'");
    Diagnostic (
     ("test", Ln: 1, Col: 2),
     "expected expression after (")])
```

Nice! We get _two_ diagnostics for the last one: one for the missing expression body, and one for the closing `)`.

The last place we need to handle failure is _at the root level_ of the parser. If the parser encounters something that doesn't look much like an expression at all we need a way to record that and skip over it. To do this we introduce a new `error` parser. Its job is to skip over a single character, and record a diagnostic:

```fsharp
let error (s: CharStream<State>) =
    match s.ReadCharOrNewline() with
    | EOS -> Reply(ReplyStatus.Error,
                   expected "valid expression character")
    | ch ->
        sprintf "Unexpected character %c" ch
        |> s.UserState.EmitDiagnostic s.Position
        Reply(Error)
```

This is a little more complex because we need to handle the case that there are no characters left to skip over. In that case the parser should return `ReplyStatus.Error` so that FParsec can move on and match the end of file. In other cases we create a diagnostic message and return our `SyntaxNode.Error` value.

We add this in to our root parser as the _last_ value in the choice. Because FParsec considers each choice in turn _until_ one succeeds our `error` parser will only be called upon if the real parsers weren't able to make any progress.

```fsharp
let parser = many (expr <|> error) .>> eof
```

With all that in place let's try parsing our original malformed expression again:

```
'(*1)+2)' ~>
  ([Product (Error,Value 1L); Error; Value 2L; Error],
   [Diagnostic (("test", Ln: 1, Col: 8),
                "Unexpected character )");
    Diagnostic (("test", Ln: 1, Col: 5),
                "Unexpected character )");
    Diagnostic (("test", Ln: 1, Col: 2),
                "missing closing ')'");
    Diagnostic (("test", Ln: 1, Col: 2),
                "expected expression after (")])
```

Much better. We get _some_ kind of parse tree at least, and the diagnostics point to the problems with the missing expression at 1:2. Where we fall down however is with the diagnostics for the missing _opening_ `(`. This is as far as we come with this example though. Introducing these recovery expressions is as much an art as it is a science.

The full source code for these parsers is available [on GitHub](https://github.com/iwillspeak/flop/tree/c5ac549363bf93897fa572caab9eff37037ee7e8).


 [fparsec]: http://www.quanttec.com/fparsec