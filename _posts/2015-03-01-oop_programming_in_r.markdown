---
layout: post
title: OOP Programming in R
date: 2015-03-01
categories: programming r oop
---

At work I have been figuring out how to write a simple simulator in R. Previously I had
done only very limited work in R, so I was surprised by all the goodies R had to offer. I
 recommend using the R studio. Even though I am longtime vim user and I prefer it at
almost all cases, I do admire the ease of managing plots, files, and various help tools
in a single application. The feature I love the most are the R vignettes. With the help
of the libraries knitr and rmarkdown R studio includes an ipython-notebook look-a-like.

What I had a hard time dealing with was to make my code object oriented. My simulator
stores its basic state in a three-dimensional matrix. Without OOP I had to include my
follow-up state in each function call.

{% highlight r %}
process <- function(state) {...}

results <- process(state)
{% endhighlight %}

Even worse while writing the simulator intermediate results had to stored be as well and
used in the evaluation step. The parameter list increased steadily.

{% highlight r %}
calc_metric1 <- function(state, results) {...}
calc_metric2 <- function(state, results) {...}
{% endhighlight %}

It took me quite some time to understand, that R has not one, but three different OOP
paradigms. I will introduce them using a point object as an example.


### S3 - conventional OOP

In S3 you basically have to tell R that a conventional object, such as a vector with two
entries is now a new class. IT 

{% highlight r %}
point1 <- c(1.0, 2.0)
class(point1) <- "point2d"
point2 <- c(26.0, 17.0)
class(point2) <- "point2d"
{% endhighlight %}

Now you can write a distance function. It will be able to deal with the different
dimensions of the points:

{% highlight r %}
dist <- function(p1, p2) {
    UseMethod("dist", p1, p2)
}

# I assume the default is the 1d case.
dist.default <- function(p1, p2) {
    return(p2 - p1)
}
dist.point2d <- function(p1, p2) {
    return(sqrt((p2[1] - p1[1])**2 + (p2[2] - p1[2])**2))
}
dist.point3d <- function(p1, p2) { ... }

dist(point1, point2)
{% endhighlight %}

Similarly the plotting and printing functions could be overloaded by including new
class sensitive ones:

{% highlight r %}
print.point2d <- function(...)
plot.point2d <- function(...)
{% endhighlight %}

### S4 - formal OOP

The S4 paradigm shocked me by its cumbersomeness. But see for yourself.
First you define the class:

{% highlight r %}
Point <- setClass(
    "Point",
    # entries of the class
    slots = c(
        x = "numeric",
        y = "numeric",
        desc = "character"
    ),
    # optional default values
    prototype = list(
        x=0.0,
        y=0.0,
        desc=""
    ),
    # optional validatation
    validity=function(object)
    {
        if((object@x < 0) || (object@y < 0)) {
            return("A negative number for one of the coordinates was given.")
        }
        return(TRUE)
    }
)
{% endhighlight %}

Next we define a function. Here we first need to reserve the name,
{% highlight r %}
setGeneric(name="set_point",
    def=function(obj, a, b) {
        standardGeneric("set_point")
    }
)
{% endhighlight %}

and only then can we define the actual function

{% highlight r %}
setMethod(f="set_point",
    signature="Point",
    definition=function(obj, a, b)
    {
        obj@x <- a
        obj@y <- b
        return(obj)
    }
)
{% endhighlight %}

I was directly diverted by this unnecessary bulk of writing and so I was relieved, when I
heard that there is similar but simpler way.

### Reference Classes

Again we define the class similar to S4:

{% highlight r %}
point <- setRefClass(
    "Point2",
    fields=c("x","y","desc"),
    prototype=list(
                x=0.0,
                y=0.0,
                desc=""
    )
)
{% endhighlight %}

But then the code for the methods is much shorter: 

{% highlight r %}
point$methods(list(
    add = function(p2) {
        x <<- x+p2$x
        y <<- y+p2$y
    },
    dist( = function(p2) {
        return(sqrt((p2$x - x)**2 + (p2$y - y)**2))
    }
))

p <- point(1, 10)
p2 <- point(30, 22)

p$dist(p2)
> [1] 23.25941
{% endhighlight %}

For even more functionalities, the R6 classes have been defined. I am a bit suspicious
about their backwards compatibility, so I will not go deeper into their structure, but
the interested reader can find more in the sources:


### Sources:

http://www.cyclismo.org/tutorial/R/s3Classes.html  
http://www.cyclismo.org/tutorial/R/s4Classes.html  
http://adv-r.had.co.nz/R5.html  
http://cran.r-project.org/web/packages/R6/vignettes/Introduction.html  
