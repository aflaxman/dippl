---
layout: lecture
title: WebPPL
description: A small probabilistic language embedded in Javascript.
---


WebPPL (pronounced 'web people'), is a small probabilistic programming language built on top of a (purely functional) subset of Javascript. This page documents the language and provides some simple examples.

# The language

## A subset of Javascript

Following the notation from the [Mozilla Parser API](https://developer.mozilla.org/en-US/docs/Mozilla/Projects/SpiderMonkey/Parser_API) our language consists of the subset of Javascript that can be built from the following syntax elements:
Program, BlockStatement, ExpressionStatement, ReturnStatement, EmptyStatement, IfStatement, VariableDeclaration, Identifier, Literal, FunctionExpression, CallExpression, ConditionalExpression, ArrayExpression, MemberExpression, BinaryExpression, UnaryExpression.

Note that there are no AssignmentExpressions or looping constructs. This is because a purely functional language is much easier to transform into Continuation Passing Style (CPS), which the WebPPL implementation uses to implement inference algorithms such as Enumeration and Particle Filtering.
While these restrictions mean that common Javascript programming patterns aren't possible, this subset is still universal, because we allow recursive and higher-order functions. It encourages a functional style that is pretty easy to use (once you get use to thinking in a functionally!).

Here is a (very boring) program that uses most of the syntax we have available:

~~~~
var foo = function(x) {
  var bar = Math.exp(3)
  var baz =  x==0 ? [] : [Math.log(bar), foo(x-1)]
  return baz
}

foo(5) 
~~~~

### Using Javascript libraries

Functions from the Javscript environment that WebPPL is called from can be used in a WebPPL program, with a few restrictions. First, these external functions must be deterministic and can carry no state from one call to another. (That is, the functions must be 'referentially transparent': calling obj.foo(args) must always return the same value when called with given arguments.) Second, external functions must be invoked as the method of an object (indeed, this is the only use of object method invocation currently possible in WebPPL). So the use of `Math.log()` in the above example is allowed: it is a deterministic function invoked as a method of the `Math` object (which is a standard object in the Javascript global environment).

Functions defines in the top level of a WebPPL program also become available in the Javascript environment. So we can, for instance, use the above function `foo` without re-declaring it:

~~~
foo(3)
~~~


## With random sampling

WebPPL is not just a subset of Javascript: is is a subset augmented with the ability to represent and manipulate *probability distributions*. Elementary Random Primitives (ERPs) are the basic object type that represents distributions. Under the hood an ERP `e` has a method `e.sample` that returns a sample from the distribution, a method `e.score` that returns the log-probability of a possible sampled value, and (optionally) a method `e.support` that returns the support of the distribution. However, these methods should not be called directly -- in order for inference operators (described later) to work ERPs should be used through the WebPPL keywords `sample`, `factor`, and so on. You can sample from an ERP with the `sample` operator. For example, using the built-in ERP `bernoulli`:

~~~~
sample(bernoulliERP, [0.5])
~~~~

There are a set of pre-defined ERPs including `bernoulliERP`, `randomIntegerERP`, etc. (Since `sample(bernoulliERP, [p])` is very common it is aliased to `flip(p)`. Similarly `randomInteger`, and so on.) It is also possible to define new ERPs directly, but most ERPs you will use will be either built in or built as the marginal distribution of some computation, via inference functions (see below).

With only the ability to sample from primitive distributions and do deterministic computation, the language is already universal! This is due to the ability to construct *stochastically recursive* functions. For instance we can define a geometric distribution in terms of a bernoulli:

~~~
var geometric = function(p) {
    return flip(p)?1+geometric(p):1
}

geometric(0.5)
~~~


## And inference

WebPPL is equipped with a variety of implementations of *marginalization*: the operation of normalizing a (sub-)computation to construct the marginal distribution on return values. These marginalization functions (which we will generally call inference functions) take a random computation represented as a function with no arguments and return an ERP that captures the marginal distribution on return values. How they get this marginal ERP differs between inference functions, and is the topic of most of most of the [tutorial](../index.html).

As an example consider a simple binomial distribution: the number of times that three fair coin tosses come up heads:

~~~
var binomial = function(){
    var a = sample(bernoulliERP, [0.5])
    var b = sample(bernoulliERP, [0.5])
    var c = sample(bernoulliERP, [0.5])
    return a + b + c}

var binomialERP = Enumerate(binomial)

binomialERP
~~~

Notice that the distribution on return values from `binomial()` and `sample(binomialERP)` are the same -- but `binomialERP` has already collapsed out the intermediate random choices.

What if we wanted to adjust the above `binomial` computation to favor executions in which `a` or `b` was true? The `factor` keyword re-weights an execution by adding the given number to the log-probability of that execution. For instance:

~~~
var funnybinomial = function(){
    var a = sample(bernoulliERP, [0.5])
    var b = sample(bernoulliERP, [0.5])
    var c = sample(bernoulliERP, [0.5])
    factor( (a|b) ? 0 : -10)
    return a + b + c}

var funnybinomialERP = Enumerate(funnybinomial)

sample(funnybinomialERP)
~~~

It is easier to build useful models (that, for instance, condition on data) with `factor`. But `factor` by itself doesn't do anything -- it interacts with *marginalization* functions that normalize the computation they are applied to. For this reason running a computation with `factor` in it at the top level -- that is, not inside a marginalization operator -- results in an error. Try running `funnybinomial` directly....

WebPPL has several inference operators, including `Enumerate` and `ParticleFilter`. These are all implemented by providing a co-routine that receives the current continuation at `sample` and `factor` statements. We explain these ideas and techniques in the next few sections.

# Examples



 