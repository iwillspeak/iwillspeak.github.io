---
title: Top-down Operator Precedence Parsing in Rust
layout: post
published: true
---

Following on from from [creating a tokeniser][tokeniser] the next step is writing a parser. The simplest way to create a hand-rolled parser is using [recursive decent][rd-wiki]. In this parsing method each non-terminal in the grammar is converted into a function which encodes it’s parsing logic. This is great for parsing things like function definitions, variable declarations and other parts of a language which can be easily recognised from their prefix. Where it falls down a bit however is parsing expressions with infix operators. This is where [Pratt operator precedence parsing][opp-wiki] comes in.

To parse multiple operator precedence lives in standard recursive decent you must left-factor your grammar. This results in one non-terminal per precedence level. To parse a literal in such a grammar the parser must descend through each precedence level again and again. This can result in quite a lot of overhead for large expressions and languages with many precedence levels in their grammar.

The solution that Pratt came up with was to parse all the consecutive operators at the current precedence level before descending to the next one. The algorithm itself is quite simple:

    fn expression(tokens, rbp) {
        let left = tokens.next().nud(tokens);
        while (next_binds_tighter_than(tokens, rbp)) {
            left = tokens.next().led(tokens, rbp);
        }
        left
    }

When parsing an expression we first ask the current token to parse its _null denotation_. This basically parses just the current literal value or variable reference. We then inspect the next token in the stream. If the next token has a binding power greater than ours then we ask it to parse it’s _left denotation_. For a simple binary operator this involves taking the operator and it’s following expression from the token stream and returning a binary operator expression. Once we no longer have any tokens in the stream that bind higher than our precedence level we return the expression we have parsed.

One of the key ideas here is that, as in [Douglas Crockford’s implementation][crockford], each token is responsible for the logic required to parse itself and any following tokens which make up the expression that it is part of. Let’s start with a simple grammar with two operators at different precedence levels and numeric literals:

{%highlight rust%}
#[derive(Clone)]
pub enum Token {
    Add, Mull, Num(i64)
}
{%endhighlight%}

The first step is to define the left binding power for each of these tokens. The higher the precedence the tighter the operator binds:

{%highlight rust%}
impl Token {
    fn lbp(&self) -> u32 {
        match *self {
            Token::Add => 10,
            Token::Mull => 20,
            _ => 0
        }
    }
}
{%endhighlight%}

The next step is to implement the `nud` method. We only have one value type in this grammar so it’s pretty simple.

{%highlight rust%}
    fn nud(&self) -> Result<Expression,String> {
        match *self {
            Token::Num(i) => Ok(Expression::Num(i)),
            _ => Err("expecting literal".to_string())
	    }
    }
{%endhighlight%}

Next up the `led` method. This is responsible for parsing our binary operators.

{%highlight rust%}
    fn led(&self, parser: &mut Parser, lhs: Expression) -> Result<Expression,String> {
        match *self {
            Token::Add | Token::Mull => {
                let rhs = try!(parser.expression(self.lbp()));
                Ok(Expression::Binary(Box::new(lhs), self.clone(), Box::new(rhs)))
            }

            _ => Err("expecing operator".to_string())
        }
    }
{%endhighlight%}

Now all that’s left is to create our `Parser` and fit all of these parts together. For simplicity our parser will take an iterator of pre-made tokens. We’ll define a few helper functions  as well to aid calling the `Token`’s `nud`, `led`, and `lbp` methods.

{%highlight rust%}
pub struct Parser<'a> {
    tokens: Peekable<Iter<'a, Token>>
}

impl<'a> Parser<'a> {

    fn new(tokens: Iter<'a, Token>) -> Self {
        Parser{ tokens: tokens.peekable() }
    }
    
    fn next_binds_tighter_than(&mut self, rbp: u32) -> bool {
        self.tokens.peek().map_or(false, |t| { t.lbp() > rbp })
    }

    fn parse_nud(&mut self) -> Result<Expression,String> {
        self.tokens.next().map_or(Err("incomplete".to_string()), |t| {
            t.nud()
        })
    }

    fn parse_led(&mut self, expr: Expression) -> Result<Expression,String> {
        self.tokens.next().map_or(Err("incomplete".to_string()), |t| {
            t.led(self, expr)
        })
    }
{%endhighlight%}

Next up is the main `expression` function. This isn’t too far from the pseudo-code we started with:

{%highlight rust%}
fn expression(&mut self, rbp: u32) -> Result<Expression,String> {
    let mut left = try!(self.parse_nud());
    while self.next_binds_tighter_than(rbp) {
        left = try!(self.parse_led(left));
    }
    Ok(left)
}
{%endhighlight%}

And that’s it. We’re all done. For a full code listing check out [the Gist][gist].



[opp-wiki]: https://en.wikipedia.org/wiki/Pratt_parser
[rd-wiki]: https://en.wikipedia.org/wiki/Recursive_descent_parser
[tokeniser]: http://willspeak.me/2015/04/15/a-rusty-guid-to-tokenising-by-hand.html
[crockford]: http://javascript.crockford.com/tdop/tdop.html
[gist]: https://gist.github.com/iwillspeak/70e0a6f74a634e22da5df8c13b1f08fe