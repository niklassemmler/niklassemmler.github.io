---
layout: post
title: "99 Problems with Scala"
date: 2016-08-07
categories: scala, data structures
---

Developing a Trie to store volume associated with TCP/UDP ports or IP addresses.
For now I am focusing on a binary Trie (basically a binary Trie, where every
node is associated with a key and a state, at least a count).


# Writing a Trie in scala

Searched for a number of templates (that's why you might find traces of other
implementations in this one), but I could not find one that fits my needs.
Therefore I implemented my own.

## Trie Data Structure

For the Trie I assume three type of nodes: Empty (an empty Trie), Leaf (a leaf
node) and Node (an intermediate node).

{% highlight scala %}
class Trie
case object Empty extends Trie
case class Leaf extends Trie
case class Node extends Trie
{% endhighlight %}

In the above code you can find two constructs typical for Scala. First, the
empty Trie Empty is defined as an *object*, which is Scala's static class. By
using an object instead of a regular class, it should be clear that no state is
kept for this empty (sub-)Trie. 

The second thing is the [*case class*](http://docs.scala-lang.org/tutorials/tour/case-classes.html).
This class makes it possible to use the class constructor for pattern matching.
An example follows:

{% highlight scala %}
def doStuffOnTrie(sth : Trie) = sth match {
  case Node => // handle Node and recurse down the Trie
  case Leaf => // handle Leaf
  case Empty => //handle Empty
}
{% endhighlight %}

Here we can define the code for each class separately, while still keeping it at
one central place.

So far we have only created empty data structures, but of course we want to
store data in them. In each non-empty node of the Trie we want to store a Key
and a count. As we might want to use the Trie for different keys (ports, IP
addresses, etc.), we will use generics:

{% highlight scala %}
class Trie[A]
case object Empty extends Trie[Nothing]
case class Leaf[A](k: A, v: Int) extends Trie[A]
case class Node[A](k: A, v: Int, l: Trie[A], r: Trie[A]) extends Trie[A]
{% endhighlight %}

You can see how we define Node slightly different to Leaf, as we have to add the
left and right subTrie. That is enough for the Trie so far. Let's focus on the key.

## Key Data Structure

So I created the following type hierarachy:

abstract class Key
class IP extends Key
class Port extends Key

Now in the code of the Trie I wanted to compare two keys (e.g. key1 < key). To
make that happen I let Key extend Ordered

{% highlight scala %}
abstract class Key extends Ordered[Key]
class IP extends Key {
    def compare(that : Key) : Int = ...
}
class Port extends Key {
    def compare(that : Key) : Int = ...
}
{% endhighlight %}

Unfortunately this introduces the unwanted side effect that IP and Port are becoming comparable with each other. To circumvent this, I change the structure as follows:

{% highlight scala %}
abstract class Key[A] extends Ordered[A]
class IP extends Key[IP] {
    def compare(that : IP) : Int = ...
}
class Port extends Key[Port] {
    def compare(that : Port ) : Int = ...
}
{% endhighlight %}

Nice, IPs are only comparable with IPs and Ports with Ports, but both are keys.
Great so lets do something with it:

{% highlight scala %}
def useKeys(key : Key) = ...
{% endhighlight %}

This doesn't compile. Instead I receive the error message "Type Key takes a type
parameter." So now I have to add the type parameter as such:

{% highlight scala %}
def useKeys[A](key : Key[A]) = ...
{% endhighlight %}

And yay, it finally compiles.

## A Key is not a Key

To use functions we have specified 

Ok, so now we have keys. However in a Trie some keys, namely keys of
intermediate nodes, are containing other keys, keys of intermediate and leaf
nodes. Only keys of leaf nodes do not contain other keys. To differentiate I
added the mix-in container.

{% highlight scala %}
trait Container[A] {
  def contains(key : A): Boolean // see if key is contained
  def commonKey(key : A) : A // find the minimal part both Keys agree on
}
abstract class Key[A] extends Ordered[A]
class IP extends Key[IP] {
    def compare(that : IP) : Int = ...
}
class Prefix extends IP with Container[IP]
{% endhighlight %}

This gives us more power to distinguish the type for each Trie node. We now can
add this information to the Trie and while we are at it, we can also specify
that all keys must be subtypes of Key

{% highlight scala %}
class Trie[A <: Key]
case object Empty extends Trie[Nothing]
case class Leaf[A <: Key](k: A, v: Int) extends Trie[A]
case class Node[A <: Key](k: A with Container[A], v: Int, l: Trie[A], r: Trie[A]) extends Trie[A]
{% endhighlight %}

Now we have a parametrized Trie, which we can use to create an IP Trie or a Port
Trie. Let's do something more useful next.

## Binary Search

Let's insert a key, count pair into an existing Trie. For that purpose we walk
from the top through the Trie, comparing the key with every node we encounter.
We add the count to every node we pass. We stop when we reach a leaf, an empty
node or an intermediate node that does not contain our key.

I follow the functional style here and assume immutable state. Therefore I
recreate the tree while walking down.

{% highlight scala %}
def insertCount[A <: Key[A]](node : Trie[A], key : A, count : Int) : Trie[A] = node match {
  case Empty => Empty // If the Node is empty
  case l : Leaf[A] => Leaf(l.k, l.v + count)
  case n : Node[A] => {
    if (!n.k.contains(key)) n
    else
      if (key <= n.k) Node(n.k, n.v+count, insertCount(n, n.v + count, insertCount(n.l, key, count), n.r)
      else Node(n.k, n.v+count, insertCount(n, n.v + count, n.l, insertCount(n.l, key, count))
  }
}
{% endhighlight %}

That reads simple enough, right? And yet it does not compile. The problem is
that we compare a two variables of type `Container[A]` and `A`.

Next let's insert a new key. This adds the difficulty, that we might need to
split an existing node when it does not contain the key:

{% highlight scala %}
def insertKey[A <: Key](node : Trie[A], key : A) : Trie[A] = node match {
  case Empty => Empty // If the Node is empty 
  case l : Leaf => insertImNode1(l, key)
  case n : Node[A] => {
    if (!n.k.contains(key)) insertImNode(n, key)
    else
      if (key<=n.k) Node(n.k, n.v+count, insertKey(node, insertKey(n.l, key, count), n.r)
      else Node(n.k, n.v+count, insertKey(node, n.l, insertKey(n.l, key, count))
  }
}
{% endhighlight %}

So far so easy, let's implement the function *insertImNode* next.

{% highlight scala %}
/* insert an intermediate node */
def insertImNode[A](n : Trie[A], k : A) : Trie[A] = {
  val commKey : A = n.commonKey()
  n match {
    case l: Leaf => {
      if (key <= commKey)
        Node(commKey, n.v, Leaf(key, 0), n)
      else
        Node(commKey, n.v, n, Leaf(key, 0))
    }
    case n : Node (_, _, )
   case n: Node(_, _, Leaf | Node, Empty)
}
{% endhighlight %}

Only works if the Node is empty

This only takes care of the case where 

100 would contain all values in between 90 to 100

In an IP a prefix 192.168.0.0/16 contains all IPs between 192.168.0.0 and
192.168.255.255. 
