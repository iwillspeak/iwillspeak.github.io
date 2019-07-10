---
title: LISP in Two Days with Rust
layout: post
published: true
---

As a sidetrack from the development of [my programming language][ullage] I've spent some time developing a LISP. The plan is to use the language as a testing ground for experimentation with transforming an AST in Rust. The syntax of LISP is simple and was developed to be easy to parse. I figured it would make a good starting point for an experimental compiler.

The language I'll be developing here is heavily inspired by the [lispy Scheme derivative][lispy]. Some elements of behaviour are cribbed directly from [Emacs LISP][elisp] as it was the LISP implementation I had to hand. By the end we should have a language capable of evaluating expressions like the following:

```lisp
(begin
	(define foo 1007)
	(define bar 330)
	(print (+ foo bar))) ; ~> prints 1337
```

## Step One - Read

The first step on the road to any programming language is parsing. We will begin by splitting the job of recognising source code like `(+ 1 foo)` into two parts. Divide and conquer! The first step is to split the string into a list of *tokens*. This step is called *lexical analysis* in the fancy pants programming language world, where each of the tokens are called *lexemes*. Tokens is good enough for us though and easier to type. We'll call it a *tokeniser*.

Our tokeniser works logically by running a set of regular expressions over the source text and choosing the token kind based on which expression matched. For our use case though we don't need a full regex engine and can make do with a [state machine][state-machines] built from a few `match` statements. For a more in-depth discussion check out [my previous blogpost on this][tokenising].

The tokens we need to recognise for LISP are pretty simple:

 * `(` and `)` for grouping [S-expressions][sexpr]
 * `[0-9]+` for numbers
 * `\n`, `\t`, ` ` etc. for whitepsace
 * Everything else is an identifier

We begin by defining the states that our tokeniser can be in. In Rust this is best modelled as an `enum`:

```rust
enum TokeniseState {
    Start,
    Lparen,
    Rparen,
    Number,
    Symbol,
    Whitespace,
}
```

The signature of the `tokenise` function is pretty simple. It accepts a borrowed string and returns a vector of tokens.

```
fn tokenise(source: &str) -> Vec<ast::Token>
```

It's probably time to define what a token is. It might be tempting at first to model a token as an `enum` too. You can do this but it sometimes becomes cumbersome. There's often state that's shared between _all_ tokens. To solve this we split the token into two parts. First a `kind` which is an `enum` containing the token's type along with any token-type-specific data such as the value of a `Number` token. The rest of the common fields are collected along with the `kind` into a `struct`:

```rust
#[derive(Debug, PartialEq)]
pub enum TokenKind {
    LeftBracket,
    RightBracket,
    Number(i64),
    Symbol(String),
}

#[derive(Debug, PartialEq)]
pub struct Token {
    pub kind: TokenKind,
    span: Span<ByteIndex>,
}
``` 

Note that the `kind` is public to allow for ergonomic `match`ing over tokens later. We're using the [`codespan`][] crate for source position and span information.

With all that in place it's time to tackle the body of the `tokenise` method. We `loop` collecting tokens and do a two-level dispatch, first on our current state then on the next character:

```rust
loop {
    let mut state = Start;
    let mut end = start;

    for c in source[start..].chars() {
        let next = match state {
            Start => match c {
                '(' => Some(Lparen),
                ')' => Some(Rparen),
                '0'...'9' => Some(Number),
                'a'...'z' => Some(Symbol),
                c if c.is_whitespace() => Some(Whitespace),
                _ => None,
            },
            Lparen | Rparen => None,
            Number => match c {
                '0'...'9' => Some(Number),
                _ => None,
            },
            Symbol => match c {
                'A'...'Z' | '0'...'9' => Some(Symbol),
                _ => None,
            },
            Whitespace => {
                if c.is_whitespace() {
                    Some(Whitespace)
                } else {
                    None
                }
            }
        };

        // If we transitioned then accept the character by moving
        // on our `end` index.
        if let Some(next_state) = next {
            state = next_state;
            end += c.len_utf8();
        } else {
            break;
        }
    }

    let token_str = &source[start..end];
    let span = Span::new(start, end);

    start = end;

    let kind = match state {
        Start => break,
        Lparen => ast::TokenKind::LeftBracket,
        Rparen => ast::TokenKind::RightBracket,
        Number => ast::TokenKind::Number(token_str.parse().unwrap()),
        Symbol => ast::TokenKind::Symbol(token_str.into()),
        // Skip whitespace for now
        Whitespace => continue,
    };

    result.push(ast::Token::with_span(
        kind,
        span.map(|s| ByteIndex(s as u32 + 1)),
    ));
}
```

Here I've elided some of the characters that can make up symbols for readability.

This should transform an input text such as `(+ 1 foo)` into a list of tokens:

```rust
vec![
	Token { kind: TokenKind::LeftBracket, ... },
	Token { kind: TokenKind::Symbol("+"), ... },
	Token { kind: TokenKind::Number(1), ... },
	Token { kind: TokenKind::Symbol("foo"), ... },
	Token { kind: TokenKind::RightBracket, ... },
]
```

The next step is to take the list of tokens and construct a syntax tree. Each node in the tree is represented by, you guessed it, an `enum`. In LISP there are two kinds of nodes in a syntax tree: *atoms*, and *forms*. The atoms in our LISP are `Number`s and `Symbol`s. Forms are parameterised S-expressions containing a list of forms or atoms. We could make do with a syntax tree node with just these two members. Our language has two *special forms* however: `if` for conditional execution, and `define` to introduce variable bindings. We will model these strongly in our tree too.

```rust
#[derive(Debug, PartialEq)]
pub enum Expr {
    /// A direct reference to a variable symbol
    Symbol(Token, String),
    /// A numeric literal
    Number(Token, i64),
    /// A conditional expression
    If(Token, Token, Box<Expr>, Box<Expr>, Box<Expr>, Token),
    /// A variable declaration
    Define(Token, Token, Token, Box<Expr>, Token),
    /// A funciton call expression
    Call(Token, Token, Vec<Expr>, Token),
}
``` 

Note that we're including all the tokens in the tree, not just the ones which have important values. This is so we have more position information available later when running linters or syntax formatters. This technique, called full-fidelity trees, is used by many compilers such as C#'s Roslyn compiler and the Swift compiler. If we were to produce an IDE integration at a later date this extra metadata would prove invaluable.

We can recognise LISP with just a single token of lookahead. We will use the Rust `Peekable` iterator for this. To begin with the peekable list of tokens is all that our parser needs:

```rust
struct ParseState<I: Iterator<Item = ast::Token>>(std::iter::Peekable<I>);
```

Our tope level parse function examines the next token. If it is an atom then we parse it directly, otherwise we delegate to `parse_form` to read the body of a form.

```rust
fn parse_expr(&mut self) -> ast::Expr {
    if let Some(token) = self.0.next() {
        use ast::TokenKind::*;
        match token.kind {
            LeftBracket => self.parse_form(token),
            RightBracket => panic!("unexpected token!"),
            Number(n) => ast::Expr::Number(token, n),
            Symbol(ref s) => {
                let sym = s.clone();
                ast::Expr::Symbol(token, sym)
            }
        }
    } else {
        panic!("invalid expression.")
    }
}
```

It's a bit `panic!`y. A production grade parser would recover more gracefully from those errors either by returning a `Result`, or by buffering a diagnostic and stubbing out the missing tokens. The latter approach allows the parser to extract _some_ syntactic information from any source text. This is becoming a far more common approach with the continued rise of the IDE.

```rust
fn parse_form(&mut self, open: ast::Token) -> ast::Expr {
    use ast::TokenKind::*;
    match self.0.peek() {
        Some(&ast::Token {
            kind: Symbol(ref sym),
            ..
        }) => match &sym[..] {
            "if" => {
                let if_tok = self.0.next().unwrap();
                let cond = self.parse_expr();
                let if_true = self.parse_expr();
                let if_false = self.parse_expr();
                let close = self.0.next().unwrap();
                ast::Expr::If(
                    open,
                    if_tok,
                    Box::new(cond),
                    Box::new(if_true),
                    Box::new(if_false),
                    close,
                )
            }
            "define" => {
                let define_tok = self.0.next().unwrap();
                let sym_tok = self.0.next().unwrap();
                let value = self.parse_expr();
                let close = self.0.next().unwrap();
                ast::Expr::Define(open, define_tok, sym_tok, Box::new(value), close)
            }
            _ => {
                let sym_tok = self.0.next().unwrap();
                let mut args = Vec::new();
                while let Some(token) = self.0.peek() {
                    if token.kind == RightBracket {
                        break;
                    }
                    args.push(self.parse_expr());
                }
                let close = self.0.next().unwrap();
                ast::Expr::Call(open, sym_tok, args, close)
            }
        },
        _ => panic!("invalid expression"),
    }
}
``` 

The `parse_form` method takes a similar approach. It attempts to mach one of our known special forms falling back to recognising a generic `Call` expression. With all this in place we can write a high-level `parse` method that our REPL will use to read source text into `ast::Expr`s:

```rust
pub fn parse(source: &str) -> ast::Expr {
    let tokens = tokenise(source);
    ParseState(tokens.into_iter().peekable()).parse_expr()
}
```

## Step Two - Eval

Now we have a syntax tree parsed it's time to move on to evaluating it. Our `eval` method will take an `Expr` and produce a `Value`. Because evaluation can encounter errors we return a `Result` type:

```rust
#[derive(Debug, PartialEq, Copy, Clone)]
pub enum Value {
    Number(i64),
    Callable(Callable),
    Nil,
}

type Callable = fn(Vec<Value>) -> EvalResult;

pub struct EvalError(String);

pub type EvalResult = Result<Value, EvalError>;

pub fn eval(expr: ast::Expr) -> EvalResult {
    eval_with_env(expr, &mut make_global_env())
}
```

Values can contain either a number or a callable function. We will not only return these as the result of evaluation but also store them in our environment.

There's not much code in `eval`, but two interesting things are going on. First we call `make_global_env` to create a new global environment. The main evaluation takes place in `eval_with_env`.

The evaluation environment for our LISP is just a `HashMap` of `Symbol` to `Value`. The `make_global_env` function just creates a new map and inserts the functions we want to be globally visible:

```rust
pub fn make_global_env() -> HashMap<String, Value> {
    let mut env = HashMap::new();

    env.insert(
        "begin".into(),
        Value::Callable(|values| Ok(last_or_nil(values))),
    );
    env.insert(
        "+".into(),
        Value::Callable(|values| {
            let mut sum = 0;
            for value in values.iter() {
                sum += value.into_num();
            }
            Ok(Value::Number(sum))
        }),
    );
    env.insert(
        "-".into(),
        Value::Callable(|values| {
            Ok(if let Some((first, rest)) = values.split_first() {
                let mut sum = first.into_num();
                if rest.len() == 0 {
                    Value::Number(-sum)
                } else {
                    for value in rest {
                        sum -= value.into_num();
                    }
                    Value::Number(sum)
                }
            } else {
                // (-) ~> 0 ; apparently
                Value::Number(0)
            })
        }),
    );
    env.insert(
        "/".into(),
        Value::Callable(|values| {
            if let Some((first, rest)) = values.split_first() {
                let mut res = first.into_num();
                Ok(if rest.len() == 0 {
                    Value::Number(1 / res)
                } else {
                    for value in rest {
                        res /= value.into_num();
                    }
                    Value::Number(res)
                })
            } else {
                Err(EvalError("Wrong number of arguments: /, 0".into()))
            }
        }),
    );
    env.insert(
        "*".into(),
        Value::Callable(|values| {
            let res = values.iter().fold(1, |acc, v| acc * v.into_num());
            Ok(Value::Number(res))
        }),
    );

    env
}
```

With the environment taken care of evaluation is fairly simple. Number atoms evaluate to themselves. Symbols are looked up in the environment:

```rust
use ast::Expr::*;
match expr {
    Symbol(_, s) => env
        .get(&s)
        .cloned()
        .ok_or_else(|| EvalError(format!("eval: Undefined symbol {}", s))),
    Number(_, n) => Ok(Value::Number(n)),
```

If evaluates the condition expression and then chooses which branch to evaluate based on the result:

```rust
    If(_, _, cond, then, elz, _) => Ok(if eval_with_env(*cond, env)?.is_truthy() {
        eval_with_env(*then, env)?
    } else {
        eval_with_env(*elz, env)?
    }),
```

Define evaluates its expression and inserts the result into the environment hashmap:

```rust
    Define(_, _, sym, value, _) => {
        let value = eval_with_env(*value, env)?;
        let sym = to_sym(sym)?;
        env.insert(sym, value.clone());
        Ok(value)
    }
```

For function calls we search for a Callable` in the environment, evaluate all of the arguments, and make the call:
 
```rust
    Call(_, sym, args, _) => {
        let sym = to_sym(sym)?;
        match env.get(&sym) {
            Some(Value::Callable(c)) => c(args
                                          .into_iter()
                                          .map(|a| eval_with_env(a, env))
                                          .collect::<Result<Vec<_>, _>>()?),
            _ => Err(EvalError(format!("eval: Invalid function {}", sym))),
        }
    }
}
```

Since our `Expr` type is simple that's all the cases we need to deal with. This is the advantage of LISP. The syntax is small. The real power comes from the way it can be combined.

## Step Three - Print

With parsing and evaluation in place we're almost at a functioning read, execute, print loop. Printing the result of evaluation is pretty simple:

```rust
fn print(result: eval::EvalResult) {
    match result {
        Ok(value) => println!(" ~> {}", value),
        Err(error) => println!(" !! {}", error),
    }
}
```

To read in our source line we first need a prompt:

```rust
fn read() -> ast::Expr {
    let mut buff = String::new();
    print!(" > ");
    std::io::stdout().flush().unwrap();
    std::io::stdin().read_line(&mut buff).unwrap();
    parse::parse(&buff)
}
```

The final REPL is then pretty simple to construct:

```rust
let mut env = eval::make_global_env();
loop {
    print(eval::eval_with_env(read(), &mut env));
}
```

Note that the global environment is initialised before the loop so state from each REPL entry is available to following entries.

```
 > (define foo 100)
 ~> 100
 > (define bar 99)
 ~> 99
 > (+ foo (- foo bar))
 ~> 101
```

## Step Four - Experiment

With the basics in place it's time to experiment. It's easy to add new callable functions to the "standard library" by including them in the global scope. The language outlined in this post doesn't have support for user-defined functions or any form of loops. Another big LISP feature missing is the ability to `quote` and `eval` forms to turn them from source code into data and back.

 [ullage]: https://github.com/iwillspeak/ullage/
 [lispy]: https://norvig.com/lispy.html
 [elisp]: https://en.wikipedia.org/wiki/Emacs_Lisp
 [state-machines]: https://en.wikipedia.org/wiki/Deterministic_finite_automaton
 [tokenising]: https://willspeak.me/2015/04/15/a-rusty-guide-to-tokenising-by-hand.html
 [sexpr]: https://en.wikipedia.org/wiki/S-expression
 [`codespan`]: https://crates.io/crates/codespan
