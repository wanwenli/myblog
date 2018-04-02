---
layout: post
title: Differences between type systems of Ocaml & Java
excerpt: Some research done for my final year project
category: programming
---

##### Introduction

When asked the differences between the Ocaml's type system and Java's,
I realized that I did not understand them well.
Therefore, I did some research about these two type systems and
posted the result as this blog.

##### Ocaml Type System

The ML languages are statically and strictly typed.
In addition, every expression has exactly one type.
In contrast, C is a weakly-typed language:
values of one type can usually coerced to a value of any other type,
whether the coercion makes sense or not.

What is "safety"?
There is a formal definition based on the operational semantics of
the programming language,
but an approximte definition is that a valid program will never fault
because of an invalid machine operation.
All memory access will be valid.
ML guarantees safety by proving that every program that
passes the type checker can never produce a machine fault.
Another functional language,
Lisp which is dynamically and strictly typed,
guarantees its safety by checking for validity at run time.
One of the consequences is that ML has no `nil` or `NULL` values.
These values would potentially cause mechine errors if used
where a value is expected.

An Ocaml programmer may initially spend a lot of time getting the Ocaml
type checker to accept his programs.
But eventually
he will find the type checker is one of his best friends.
It helps a programmer figure out
where errors may be lurking in programs.
If a change is made, the type checker will track down the parts that
are affected.
At the meantime,
here are some rules about type checking.

* Every expression has exactly one type.
* When an expression is evaluated, one of the four things may happen:
	* it may evaluate to a value of the same type as the expression,
	* it may raise an exception,
	* it may not terminate,
	* it may exit.

One of the important points in Ocaml is that
there are no "pure commands".
Every assignment produce a value
- although the value has the trivial `unit` type.
Below let us see some examples of how Ocaml type system works:

{% highlight ocaml linenos %}
	if 1 < 2 then
		1
	else
		1.3
{% endhighlight %}

When compiling the code,
the compilor would tell you there is a type error on line 4,
characters 3-6. This is because the type checker requires that
both branches of the conditional statement have the same type
no matter how the test turns out.
Since the expressions `1` and `1.3` have different types,
the type checker generates an error.

##### A Strong Static Language: Java
Java is considered one of the most static languages,
but it implemented a comprehensive reflection API which
allows you to change classes at run time,
thus resembling more dynamic languages.
This feature enables Java Virtue Machine (JVM)
to support very dynamic languages such as Groovy.

Java needs everything to be defined so that
you know all the time what type of object you have and
whether you call them properly.
In addition,
Java does not allow code outside of a class.
It has been a major reason why people complain that
Java forces you to write too much boilerplate.

The popularity of Java and its strong adherence to strong typing
made a huge impact on the programming landscape.
Strong typing advocates lauded Java for fixing the cracks in C++.
But many programmers found Java overly prescriptive and rigid.
They wanted a fast way to write code without all of the extra definition of Java.
As a result, strong dynamic typing languages,
such as JavaScript, Python and Ruby, are becoming more and more popular
in recent years.

##### What are their differences?
Anyway,
there is no stronger or weaker type system between these two systems.
Ocaml and Java are both to static and strong typing languages.
They both have primitive, array and class types.
(The standard ML language does not support class types but 
Objective-caml does.)
However Java and Ocaml do have some differences in their type systems.
Ocaml is a functional language and
a function can be passed as a argument,
known as higher order functions.
The type of a function is defined by both its input and output type.
For example in the chunck of code below,

{% highlight ocaml linenos %}
	let square x = x * x;
	val square : int -> int = <fun>
{% endhighlight %}

`square` is a function that takes an integer and returns an integer.
Most importantly,
square can be passed to another function as an input.
However it is not done as natural in Java.
In Java higher order function is usually done by
wrapping the function within an interface as below:

{% highlight java linenos %}
  public int methodToPass() {
		// do something
	}

	public void dansMethod(int i, Callable<Integer> myFunc) {
		// do something
	}
{% endhighlight %}

then use a inner class to call the method `methodToPass` as shown below:

{% highlight java linenos %}
  dansMethod(100, new Callable<Integer>() {
		public Integer call() {
			return methodToPass();
		}
	});
{% endhighlight %}

Next, pattern matching is something unique to Ocaml compared to Java.
A similar feature in Java is perhaps `switch` statement.
However it does not have many things to do with type system.
{% highlight ocaml linenos %}
	let rec sigma f = function
		| [] -> 0
		| x :: l -> f x + sigma f l;;
	val sigma : ('a -> int) -> 'a list -> int = <fun>
{% endhighlight %}

Last but not the least,
type inference is a major difference between Ocaml and Java type systems.
Most conventional static typing languages require programmers to restate the types of expressions.
For example,
the following Java code instantiates a method to increment an integer by 1.
{% highlight java linenos %}
int succ (int n) {
    return n + 1;
}
{% endhighlight %}
In OCaml, all programs can be written such that the types they use are competely inferred, i.e.
it is never necessary to explicitly define and declare types in OCaml.
However, defining important types in a program is a good way to leverage static type checking by providing machine-checked documentation,
improving error reporting and tightening the type system to catch more errors.

The above Java code can be re-written as
{% highlight ocaml linenos %}
let succ n = n + 1;;
val succ : int -> int = <fun>
{% endhighlight %}
OCaml infers that the function maps integers onto integers even though no types were declared.
The inference was based upon the use of the integer operator + and the integer literal 1.

##### Conclusion

In summary,
the type system of Java has lots of similarities compared to Ocaml.
Still Java is one of the dominant and most popular programming languages.
