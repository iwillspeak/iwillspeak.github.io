---
title: A Rusty Guide to Tokenising by Hand
layout: post
published: true
---

I recently wrote about [creating a tokeniser in C++ using Ragel]({% post_url 2015-03-24-tokenising-in-c-with-ragel %}). In that post I created a lexer which recognised a set of three tokens. In this post I'll discuss creating a lexer by hand in [Rust](http://rust-lang.org) to recognise the same language.

The job of a tokeniser is to break down a piece of text into a set of tokens, or lexemes. A token consists of two parts: an identifier which defines which regular expression that the string matched, and an optional value. If the token represents a number then the value might be the integer representation of that number.

We defined the language with the following three definitions:

    Var ::= [a-z][a-z0-9]*
    Num ::= [0-9]+
    Plus ::= '+'

There are four main stages to creating a tokeniser from this set of regular expressions:

1. Create a *nondeterministic finite-state automata* (NFA) which accepts each of the regular expressions.
2. Join these automata together to create a single NFA which accepts all of the regular expressions.
3. Convert the NFA to a *deterministic finite-state automata* (DFA).
4. Write a program which implements the DFA.

## NFA Construction

We will start with the simplest regular expression `'+'`. The NFA that accepts this has two states. We move from the initial state to the final state when we see `'+'`.

![Plus Graph](/img/posts/nfa-initial.dot.3.svg)

The machine that accepts `Num` tokens also has two states. The first transition from the start state ensures that at least one digit is matched. We then allow any number of additional numbers with a looping transition.

![Num Graph](/img/posts/nfa-initial.dot.2.svg)

For the final FSA which accepts `Var`s has two main parts. The first is the two-node machine which accepts `[a-z]`. This is then joined with a **&epsilon;** transition to a one-state machine which accepts the Kleene star of `[a-z0-9]*`.

![Var Graph](/img/posts/nfa-initial.dot.svg)

## NFA Composition

Once we have the NFAs for each of the individual tokens we can combine them into a single machine. We do this by adding a new start state and then adding a **&epsilon;** transition from this new state to the start state of each of the individual machines.

We have also added in a machine which accepts any number of whitespace characters. This will allow us to separate each token by optional whitespace.

![Combined NFA Graph](/img/posts/nfa-combined.dot.svg)

## NFA &rarr; DFA Conversion

To convert the NFA to a DFA we will create a DFA in which each state represents a set of NFA states.

To calculate these sets of states we compute the &epsilon;-closure of each transition in the NFA. We start with the &epsilon;-closure of the start state and repeatedly choose a transition we haven't followed and then add in a new transition going to the &epsilon;-closure of that transition.

| DFA State | NFA States | Transitions | Final |
|-|
| *S* | { `s`, `v1`, `p1`, `w1` } | `\s` -> *A* | n |
| | | `\+` -> *B* |
| | | `0-9` -> *C* |
| | | `a-z` -> *D* |
| *A* | { `w1` } | `\s` -> *A* | y |
| *B* | { `p2` } | | y |
| *C* | { `n2` } | `0-9` -> *C* | y |
| *D* | { `v2`, `v3` } | `a-z` -> *E* | y |
| | | `0-9` -> *e* |
| *E* | { `v3` } | `a-z` -> *E* | y |
| | | `0-9` -> *e* |


## Implementation!

Now all that's left to do is to write a program that implements the state machine we have constructed. We'll start by defining the three structures that we will use.

The first is the `Tok` structure. This represents each token we generate. A nice feature of Rust is that we can store the value directly in the enum along with the identifier of the token. This is really just syntactic sugar for a tagged union in c-land. There is one major advantage though. You can't read the wrong type of data out due to the type safety enforced by the Rust language.

{% highlight Rust %}
#[derive(Debug,PartialEq)]
pub enum Tok {
    Var(String),
    Num(i32),
    Plus
}
{% endhighlight %}

Next up is the state enumeration. It's a pretty simple one. One thing to note is that we make sure it implements `PartialEq` so we can compare values of it later on.

{% highlight Rust %}
#[derive(PartialEq)]
enum State {
    S, A, B, C, D
}
{% endhighlight %}

The final structure is the tokeniser itself. This stores the buffer of characters to be tokenised, represented as a `String`, and the start of the token currently being tokenised. Once we finish consuming a token we advance this index so that the next token in the stream can be recognised.

{% highlight Rust %}
pub struct Tokeniser {
    ts: usize,
    chars: String
}
{% endhighlight %}

The final state machine then loops through the characters in the buffer, starting from where it left off. If the character refers to a valid transition in the state machine it moves to that state and keeps on looking for more characters. Once we can no longer move to another state we return a token based on the last accepting state that we entered.

Seen as in this state machine we can't move to any non-accepting states after entering an accepting state we make the simplifying assumption that the last accepting state was just the last state. If the last state wasn't an accepting state then we return `None`. If the last state was an accepting state we return a `Some(Tok)` with the right data read from the slice of the buffer which we just recognised.

{% highlight Rust %}
fn next_match(&mut self) -> Option<Tok> {

    loop {
        
        let mut state = State::S;
        let mut te = self.ts;

        for c in self.chars[self.ts..].chars() {

            // find the next transition in the
            //state machine
            let next = match state {
                State::S => match c {
                    ' ' | '\t' => Some(State::A),
                    '+' => Some(State::B),
                    '0'...'9' => Some(State::C),
                    'a'...'z' => Some(State::D),
                    _ => None
                },
                State::A => match c {
                    ' ' | '\t' => Some(State::A),
                    _ => None
                },
                State::B => None,
                State::C => match c {
                    '0'...'9' => Some(State::C),
                    _ => None
                },
                State::D => match c {
                    'a'...'z' | '0'...'9' => Some(State::D),
                    _ => None
                }
            };

            // If we found a transition then consume
            // the character and move to that state
            if let Some(next_state) = next {
                state = next_state;
                te += 1;
            } else {
                break;
            }
        }

        // once we can no longer match any more
        // characters we decide what token to return
        let token_str = &self.chars[self.ts..te];

        self.ts = te;

        // If we recognised some whitespace, look for
        // the next token instead
        if state == State::A {
            continue;
        }

        // Depending on which state we're in we know
        // which token we have just accepted
        return match state {
            State::B =>
              Some(Tok::Plus),
            State::C => 
              Some(Tok::Num(token_str.parse().unwrap())),
            State::D =>
              Some(Tok::Var(token_str.to_string())),
            _ => None
        };
    }
}
{% endhighlight %}

For more code take a look at the [full Microcrate](https://gist.github.com/iwillspeak/a8a8c0f03524d8ce6d19).
