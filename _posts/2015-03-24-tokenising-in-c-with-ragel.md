---
title: Tokenising in C++ with Ragel
layout: post
published: true
---

[Ragel](http://www.colm.net/open-source/ragel/) is an alternative to Lex/Flex for creating lectures from a set of regular expressions. It is far more powerful however. Where Lex allows you to chain together a set of regular expressions and run some actions written in C or C++ in response to a match Ragel allows you to generate an arbitrary state machine by combining regular expressions and then run actions at any point in the matching of those expressions. It also supports a wide variety of 'host' languages, including C#, Ruby and Go.

We will attempt to recognise a simple set of tokens consisting of variables, numbers and the plus sign.

    Var ::= [a-z][a-z0-9]*
    Num ::= [0-9]+
    Plus ::= '+'

To recognise it we define the following Ragel machine. It uses the shorthand syntax for defining a tokeniser machine. This allows us to define a set of regular expressions actions that will be run when the expression is matched similar to the functionality of lex.

{% highlight ragel-cpp %}
%%{
  machine ExampleLexer;

  main := |*

    digit+ => {
      CAPTURE_TOKEN(Num);
      fbreak;
    };

    alpha (alpha | digit) * => {
      CAPTURE_TOKEN(Var);
      fbreak;
    };
    
    '+' => {
        CAPTURE_TOKEN(Plus);
        fbreak;
    };

    space+;
  *|;
}%%
{% endhighlight %}

Notice that we explicitly have to match whitespace and ignore it.  We will define the macro `CAPTURE_TOKEN` to expand to the C++ code we require to return a token of that type.

We will represent tokens as a simple C++ struct:

{% highlight cpp %}
struct Token
{
    enum TokenType {
        Var,
        Num,
        Plus,
        End,
        None
    };

    TokenType type;
    std::string value;
};
{% endhighlight %}

We now just need to define a simple C++ class to encapsulate the lexer's state. We define the member variables `p`, `pe` and `eof`  which Ragel needs to keep track of the state of the buffer and initialise them in the constructor. If the data was being read in chunks from a buffered source you would need to keep on updating these pointers until you reached the end of the input storem.

Since we are using a tokeniser machine we need to set up the `ts` and `te` variables which Ragel will set to point to the matched token in the buffer before calling each action.

The final set of Ragel variables (`act`, `cs`, `top`, `stack`) define the internal workings of the state machine.

{% highlight cpp %}
class Lexer
{
public:
    Lexer(std::string& input)
      : buffer(input) {

        %%write init;

        // set up the buffer here
        p = buffer.c_str();
        pe = p + buffer.size();
        eof = pe;
    }

    Token* next() {
        auto token = new Token();
    
        token->type = Token::None;
    
        do {
    
            if (cs >= ExampleLexer_first_final) {
                token->type = Token::End;
            }
    
            %%write exec;
            
            if (cs == ExampleLexer_error) {
                token->type = Token::None
                return token;
            }
            
        } while (token->type == Token::None);
    
        return token;
    }

private:
    // buffer state
    const char* p, * pe, * eof;
    // current token
    const char* ts, * te;
    // machine state
    int act, cs, top, stack[1];

    std::string buffer;
};
{% endhighlight %}

The one last thing we need to define is the macro which captures the token. This macro basically writes directly into the `token` variable defined as a local in the `next()` method.

{% highlight cpp %}
#define CAPTURE_TOKEN(t) \
  token->value = std::string(ts, te-ts); \
  token->type = Token::t
{% endhighlight %}

And there you go. All done. We can now pass a simple buffer to our lexer and keep on calling `next()` until we run out of tokens. This lecture will return the tokens one at a time as it reads them. You can have Ragel tokenise the whole input if you want by removing the `fbreak` calls from the actions and looping until you receive either `None` or `End`.

The full code for this article is in [this Gist](https://gist.github.com/iwillspeak/cd27b740d658a6f38c8c). 
