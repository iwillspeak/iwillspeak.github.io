---
title: A Rusty Guid to Tokenising by Hand
layout: post
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
