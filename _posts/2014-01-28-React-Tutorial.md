---
layout: post
title: "OCaml React package tutorial"
description: Unofficial tutorial for OCaml React package
categories: programming
---

React is an OCaml package made for [reactive programming](http://en.wikipedia.org/wiki/Reactive_programming)
using OCaml.
The objective of this post is to illustrate and explain some examples of the React API.
Therefore the basic concepts of React, such as Signals, Events and side effects, will not be repeated here.
Below are some prerequisites for reading this post.

* [OCaml](http://ocaml.org/) (version 4.00.1 or above)
* [OPAM](http://opam.ocamlpro.com/) (OCaml PAckage Manager)
* React (installation via OPAM)
* [React Introduction](http://erratique.ch/software/react/doc/React.html)

Most of the examples used in this post are inspired by
[test cases](https://github.com/dbuenzli/react/blob/master/test/test.ml)
of React project.
To report an issue in the post,
contact me through my Github account.
Thank you for your feedbacks.

##### Using React package

For compilation

```
ocamlfind ocamlopt -o clock -linkpkg \
    -package react clock.ml
```

For OCaml Toplevel

```
#use "topfind";;
#require "react";;
open React;;
```

##### Module [React.E](http://erratique.ch/software/react/doc/React.E.html)
{% highlight ocaml %}
val once : 'a React.event -> 'a React.event
{% endhighlight %}
once e is e with only its next occurence.
Any occurence after time t will not affect e.

{% highlight ocaml linenos %}
open React (* omitted in the examples below *)
let w, send_w = E.create ()
let x = E.once w
let pr_x = E.map print_int x
let () = List.iter send_w [0; 1; 2; 3]
{% endhighlight %}

The output would be only 0.
All the values after the first occurrence are omitted.

{% highlight ocaml %}
val drop_once : 'a React.event -> 'a React.event
{% endhighlight %}

drop_once e is e without its next occurrence.
Similarly, if we change the third line of the example above,
the output would be 123.
The reason is because the NEXT occurrence of e is omitted.

{% highlight ocaml %}
val app : ('a -> 'b) React.event -> 'a React.event -> 'b React.event
{% endhighlight %}

app ef e occurs when both ef and e occur simultaneously.

{% highlight ocaml %}
let f x y = x + y
let w, send_w = E.create ()
let x = E.map (fun w -> f w) w
let y = E.drop_once w
let z = E.app x y
let pr_z = E.map print_int z
List.iter send_w [0; 1; 2; 3]
{% endhighlight %}

The output would be 246.
As explained before,
drop_once makes y only take updates from the second occurrence onwards.
Variable z takes the value of x when both x and y are updated.

{% highlight ocaml %}
val map : ('a -> 'b) -> 'a React.event -> 'b React.event
{% endhighlight %}
map f e applies f to e's occurrences.

{% highlight ocaml %}
val stamp : 'b React.event -> 'a -> 'a React.event
{% endhighlight %}
stamp e v is map (fun _ -> v) e.
It fixes the occurrence of e to v.

{% highlight ocaml %}
val filter : ('a -> bool) -> 'a React.event -> 'a React.event
{% endhighlight %}
filter p e are e's occurrences that satisfy p.

{% highlight ocaml %}
val fmap : ('a -> 'b option) -> 'a React.event -> 'b React.event
{% endhighlight %}
fmap fm e are occurrences of e filtered and mapped by fm.

{% highlight ocaml %}
let v, send_v = E.create ()
let x = E.stamp v 1 (* x: 1111 *)
let y = E.filter (fun n -> n mod 2 = 0) v (* y: 02 *)
let z = E.fmap (fun n -> if n = 0 then Some 1 else None) v (* z: 1 *)
List.iter send_v [0; 1; 2; 3]
{% endhighlight %}

{% highlight ocaml %}
val diff : ('a -> 'a -> 'b) -> 'a React.event -> 'b React.event
{% endhighlight %}
diff f e occurs whenever e occurs except on the next occurence.
In another words,
it is triggered since the second the occurrence.
Occurences are f v v' where v is e's current occurrence and v' the previous one.

{% highlight ocaml %}
val changes : ?eq:('a -> 'a -> bool) -> 'a React.event -> 'a React.event
{% endhighlight %}
changes eq e is the occurrences of e with occurences equal to the previous one dropped.
Equality is tested with eq (defaults to structural equality).
The behavior is similar to Signals.

{% highlight ocaml %}
let x, send_x = E.create ()
let y = E.diff ( - ) x
let z = E.changes x
List.iter send_x [0; 0; 3; 4; 4]
{% endhighlight %}

The occurrences of y are 0310 (difference between current and previous one),
while those of z are 034 (only changes are recorded).

{% highlight ocaml %}
val dismiss : 'b React.event -> 'a React.event -> 'a React.event
{% endhighlight %}
dismiss c e is the occurences of e except the ones when c occurs.

{% highlight ocaml %}
let x, send_x = E.create ()
let y = E.fmap (fun x -> if x mod 2 = 0 then Some x else None) x
let z = E.dismiss y x
List.iter send_x [1; 2; 3; 4; 5; 6]
{% endhighlight %}
The occurrences of y are 246 (even numbers).
Occurrences of z are 135.
When y has occurrence, z will not have any occurrence.

{% highlight ocaml %}
val until : 'a React.event -> 'b React.event -> 'b React.event
{% endhighlight %}
until c e is e's occurences until c occurs.

{% highlight ocaml %}
let x, send_x = E.create ()
let stop = E.filter (fun s -> s = "c") x
let y = E.until stop x
let pr_y = E.map print_string y
List.iter send_x ["a"; "b"; "c"; "d"; "e"]
{% endhighlight %}

The output is ab.
All the occurrences after stop event are dropped,
although stop has no more new updates.

{% highlight ocaml %}
val accum : ('a -> 'a) React.event -> 'a -> 'a React.event
{% endhighlight %}
accum ef i accumulates a value, starting with i,
using e's functional occurrences.
It takes the previous occurrence, applies to the function and evaluates
the result as the current occurrence.

{% highlight ocaml %}
let f, send_f = E.create ()
let a = E.accum f 0
let pr_a = E.map print_int a
List.iter send_f [( + ) 2; ( - ) 1; ( * ) 2];
(* Output is 2 -1 -2 *)
{% endhighlight %}

{% highlight ocaml %}
val fold : ('a -> 'b -> 'a) -> 'a -> 'b React.event -> 'a React.event
{% endhighlight %}
fold f i e accumulates the occurrences of e with f starting with i.
The behavior of fold is similar to List.fold_left.

{% highlight ocaml %}
let x, send_x = E.create ()
let y = E.fold ( * ) 1 x
let pr_y = E.map print_int y
List.iter send_x [1; 2; 3; 4]
{% endhighlight %}
The output would be 1 2 6 24.

{% highlight ocaml %}
val select : 'a React.event list -> 'a React.event
{% endhighlight %}
select el is the occurrences of every event in el.
If more than one event occurs simultaneously the leftmost is taken and the others are lost.

{% highlight ocaml %}
let w, send_w = E.create ()
let x, send_x = E.create ()
let y = E.map succ w
let z = E.map succ y (* z and y are considered simultaneous *)
let t = E.select [w; x] (* t: 0 1 2 3 4 *)
let sy = E.select [y; z] (* sy: 1 3 4 5 *)
let sz = E.select [z; y] (* sz: 2 4 5 6 *)
send_w 0;
send_x 1;
List.iter send_w [2; 3; 4]
{% endhighlight %}
The expected occurrences of all variables are written in the comments.
To know more about simultaneous events, read
[this](http://erratique.ch/software/react/doc/React.html#simultaneity).

{% highlight ocaml %}
val merge : ('a -> 'b -> 'a) -> 'a -> 'b React.event list -> 'a React.event
{% endhighlight %}
merge f a el merges the *simultaneous* occurrences of every event in el using f and the accumulator a.
You may refer to the previous API to check the meaning of simultaneous events.
{% highlight ocaml %}
let w, send_w = E.create ()
let x, send_x = E.create ()
let y = E.map succ w (* y is a simultaneous event w.r.t. w *)
let z = E.merge (fun acc v -> v::acc) [] [w; x; y]
send_w 1;
send_x 4;
send_w 2;
{% endhighlight %}
The occurrences of z are [2; 1], [4], [3; 2].
Hint: fun acc v -> v::acc pushes every v to the accumulator acc.

{% highlight ocaml %}
val switch : 'a React.event -> 'a React.event React.event -> 'a React.event
{% endhighlight %}
switch e ee is e's occurrences until there is an occurrence e' on ee,
the occurrences of e' are then used until there is a new occurrence on ee, etc.

{% highlight ocaml %}
val fix : ('a React.event -> 'a React.event * 'b) -> 'b
{% endhighlight %}
fix ef allows to refer to the value an event had an infinitesimal amount of time before.

##### Module [React.S](http://erratique.ch/software/react/doc/React.S.html)

{% highlight ocaml %}
val hold : ?eq:('a -> 'a -> bool) -> 'a -> 'a React.event -> 'a React.signal
{% endhighlight %}
hold i e has the value of e's last occurrence or i if there was not any.

{% highlight ocaml linenos %}
let x, send_x = E.create ()
let a = S.hold 0 x
let pr_a = S.map print_int a
List.iter send_x [1; 2; 3; 3; 3; 4; 4; 4]
{% endhighlight %}
A 0 will printed out at line 3 because x has no occurrences by then.
Then 1234 will be printed.
Note that only updates that change the value of a signal are printed.

Many API are similar to those in React.E.
Therefore this tutorial jumps straight to Lifting combinators.
Lifting transforms a regular function to make it act on signals.
For example, l2 takes a function with two arguments as its input.

{% highlight ocaml %}
val l2 : ?eq:('c -> 'c -> bool) ->
    ('a -> 'b -> 'c) -> 'a React.signal -> 'b React.signal -> 'c React.signal
{% endhighlight %}

Example:

{% highlight ocaml %}
let f x y = x mod y
let fl = S.l2 f
val fl : int React.signal -> int React.signal -> int React.signal = <fun>
{% endhighlight %}

##### Some other examples

* [Clock.ml](https://github.com/dbuenzli/react/blob/master/test/clock.ml)
* [Breakout.ml](https://github.com/dbuenzli/react/blob/master/test/breakout.ml)
* [Oreo project](https://github.com/swwl1992/oreo) OCaml Reactive programming used on Web authored by me
