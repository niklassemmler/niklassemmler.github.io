---
layout: post
title:  "Expanding the LOC with Lisp"
date:   2015-01-21
categories: programming
---

Over the holiday season I found a door into the programming language Lisp through the very
well-written book "Land of Lisp" by Conrad Barski. Let's make one thing clear though,
Lisp's syntax is hard to get used to when coming from a pure imperative language as C or
Pascal:

* Parantheses are written around function call and arguments:

{% highlight lisp %}
> (print "Hello world!")
Hello world
{% endhighlight %}

* Lists and functions use round parantheses, data is ditinguished by a single quote 

{% highlight lisp %}
> (print '(2 3))
(2 3)

> (print (2 3))
*** - EVAL: 2 is not a function name
{% endhighlight %}

Still Lisp amazes me simply becauses it redefines the use of a single line of code. In a
usual imperative code example, you would find assignments and operations intermingled and
nested into for loops and if conditionals:

{% highlight C %}
for (int i=1; i < n; i++)
    a = input[i]
    if (not_ready(a)) { a = prepare(a) }
    c = operation((a, b)
    output[i] = c
{% endhighlight %}

Now there are of course already short cuts to this version. C for example can put the
conditionals to work:

{% highlight C %}
if ((a = input[i]) && not_ready(a)) { a = prepare(a) }
{% endhighlight %}

And python of course will move the for loop into a list comprehension:

{% highlight Python %}
[operation(x, b) for x in input]
{% endhighlight %}

The power of lisp really is to reduce the number of assignments and stretch the code to
the right.

{% highlight lisp %}
(map #'operation 
    map (lambda (in) 
            if (not_ready(in)) prepare(a))
    input)
)
{% endhighlight %}

Now of course I have used the functional primitive "map" and of course I could also have
implemented the python code with map. However then I would have had to create a lambda
function for the "operation". It is not the right question, to ask whether the same thing could be done in
Python, the question is how hard is it to do the same.

Pythons map and reduce functionality are a sledge hammer to the fine scalpel that Lisp is
with its many primitivie. Just map can be divided into at least the following three
functions:

{% highlight lisp %}
map
mapc
mapcar
{% endhighlight %}

The implicit goal of Lisp is to keep all data types as simple as possible to operate on
them with precise primitives.
