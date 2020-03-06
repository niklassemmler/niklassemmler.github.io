---
layout: post
title: DSLs in Python 
date: 2015-05-13
categories: dsl, python
---

In my master thesis I have designed a Domain Specific Language (DSL) in python.
In this article I want to give the unfamiliar reader a short introduction to
DSLs and then explain how a DSL can be designed in python.

A DSL is a programming language in one target domain.  It enables the user to
write code that solves only the domain problem and nothing more.  Therefore, a
good DSL has to be easy to write for the user and yet able to express most of
the problems that can be found in the domain. Hence it is a considerable design
challenge and requires sufficient domain knowledge.

In Python typical examples are the [pandas library][pandas] designed for data
management and the [scapy library][scapy] for analysis of network packets. A
short example of scapy follows.

{% highlight python %}
# print packet summary for all UDP packets
from scapy.all import *

def handle(packet):
	if packet[IP].proto == 17:
		print "UDP: %s" % packet.summary()

sniff(prn = lambda x: handle(x), timeout = 25)
{% endhighlight %}

Scapy and pandas are examples of embedded or internal DSLs. Embedded DSLs
exploit their host language to create domain abstractions. Thereby the structure
of the embedded DSL depends hugely on the design of the host language itself. In
Python typically exploited mechanisms are:

* constants (above: IP)
* methods (above: sniff)
* classes (above: packet and packet.summary)
* operators (+, -, &, ...)

The disadvantage of an embedded / internal DSL are the restrictions imposed by
the host language. The speed of the embedded DSLs is dependent on the host
language and the DSL can only express a subset of what can be expressed in the
host language. Furthermore some parts of the host language cannot be overloaded
(in Python statements) or removed from the users control.

The alternative is the specification of an external DSL, i.e. a DSL that is not
embedded into any host language but makes use of its own compiler. 
Advantage of external DSLs are the complete freedom in the form of the language
(it is in effect its own language) and the possibility of improved execution
time. While a DSL could potentially produce bytecode directly it is most often
used to generate code in another general purpose programming language.

On the downside a compiler has to be constructed for a DSL which is no simple
task. There are at least three steps involved in the design of a DSL:

1. Defining the basic elements - Tokens
2. Specifying syntax rules - Parser
3. Generating an output - (Code) Generator

Creating all of the above from scratch is a painful (although not completely
unrewarding) exercise. The process can be greatly simplified by using 
compiler construction tools. In the remainder of this article I will shed light
on the most prominent alternatives available in Python namely [PLY][ply] conceived by
David Beazley in 2001 and [Pyparsing][pyparsing] created by Paul McGuire in 2006.
PLY follows the older LEX and Yacc construction tools, which use a specific grammar file to define both tokens and parser rules. Pyparsing was created as an alternative to Yacc and LEX and introduces its own DSL embedded in Python.

As an example I choose a construction of a calculator that supports nothing but
addition. It should allow calculations in the form of 'foo = 3 + 5'. So in the
first step we have to define the plus and equal operator and regular expressions
for numbers and variable names.

In PLY each of the introduced tokens is assigned to a variable starting with
't\_' and a list named 'tokens' is used to refer to the tokens without the
prefix. When we call
the lexer the tokens are extracted from the context.
 
{% highlight python %}
from ply import lex

# Each of the variables defining a token starts with t_
t_ID = r'[a-zA-Z][a-z_0-9]*'
t_PLUS = r'\+'
t_NUM = r'\d+'
t_EQ = r'='

# Ignore white spaces
t_ignore = r' '

# List refers to the tokens by their name
tokens = ['ID', 'PLUS', 'NUM', 'EQ']

# Actually starting the lexer
lexer = lex.lex()
{% endhighlight %}

Pyparsing does not come of very different, only that the regular expressions are
created from building blocks. The literals ``+'' and ``='' are suppressed, so
that they do not show up in the final parse result.

{% highlight python %}
from pyparsing import Word, alphas, Literal, alphanums, nums

plus = Literal('+').suppress()
equal = Literal('=').suppress()

# alphas, alphanums, nums are regular expressions building blocks
name = Word( alphas, alphanums + "_" )
number = Word( nums )
{% endhighlight %}

Next we will create the parsing rules for the DSL. We will create an
abstract syntax tree (AST) which we can use later to generate the actual code.
(Strictly speaking this is not necessary, as the code generation could as well
be done in the parsing rules themselves, but this is not possible for more
complicated scenarios.) Parsing rules have an identifier and combine multiple of
the defined tokens or additional parsing rules. 

PLY again takes some kind of meta approach. The parsing rules are actually the
doc strings of functions starting with 'p\_' and follow the formal Backus Naur
Form a formal language for defining languages. Results are moved up from the
most basic rules to the more complicated by means of the 'p' argument. Every
function defines an operation over the elements of its parsing rules and assigns
the result to the first element of p. The parsing is executed by calling 'yacc'
from the ply package.

{% highlight python %}
from ply import yacc

def p_root(p):
    """ root : assign
             | expr """
    p[0] = p[1]

def p_assign(p):
    """ assign : ID EQ expr """
    p[0] = ('assign', p[1], p[3])

def p_expr_plus(p):
    """ expr : NUM PLUS expr """
    p[0] = ('plus', p[1], p[3])

def p_expr_num(p):
    """ expr : NUM """
    p[0] = p[1]

parser = yacc.yacc()

print(parser.parse(string))
{% endhighlight %}

In pyparsing the parser rules are defined by means of overloaded operators.
PArts of a rule are combined with '+' and alternatives are specified with
parentheses and '|'. A special operator is '<<' it enables recursion of the
contained rules.

{% highlight python %}
from pyparsing import Forward
# Parsing

def fun_plus(x, y, toks):
    return ("plus", toks[0], toks[1])

def fun_assign(toks):
    return ("assign", toks[0], toks[1])
def fun_plus(x, y, toks):
    return ("plus", toks[0], toks[1])

def fun_assign(toks):
    return ("assign", toks[0], toks[1])

expr = Forward() # allows recursion
expr2 = (expr | number)
expr << (number + plus + expr2) # recurse
expr.setParseAction(fun_plus)

assign = (name + equal + expr)
assign.setParseAction(fun_assign)

root = ( assign | expr )

print(root.parseString(string))
{% endhighlight %}

With the logic described so far we can create a tree consisting of tuples. I
leave it as an excercise to generate the code from this tree.

[pandas]: http://pandas.pydata.org/ "Python Data Analysis Library"
[scapy]: http://www.secdev.org/projects/scapy/ "Scapy"
[ply]: http://www.dabeaz.com/ply/ply.html "PLY"
[pyparsing]: pyparsing.wikispaces.com/HowToUsePyparsing "Pyparsing"

