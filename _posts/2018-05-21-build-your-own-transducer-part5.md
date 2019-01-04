---
layout: post
title: Build Your Own Transducer and Impress Your Cat - Part 5
description: Brief introduction to what transducers are and how to use them.
date: 2018-05-21
tags: Clojure, Transducer, Beginners
categories: Clojure
---

This post is a part of a serie:

1. [Introduction to transducers](2018-05-08-build-your-own-transducer-part1.md)
2. [Anatomy of a transducer](2018-05-10-build-your-own-transducer-part2.md)
3. [Stateful transducers](2018-05-12-build-your-own-transducer-part3.md)
4. [Early termination in transducers](2018-05-13-build-your-own-transducer-part4.md)
5. Functions which are using transducers (this post)
6. [Transducer exercises and solutions](https://github.com/green-coder/transducer-exercises)

---

This article describes some functions which are using transducers, and how to write your own.

# Transducers, getting `into` using them

If you followed the previous 4 parts of this blog serie, you may have noticed that I only mentioned one function which is using transducers so far: the `into` function.

``` clojure
(into [] (map inc) (list 3 4 5))
; => [4 5 6]
```

There are more of that kind. Their role is to provide the context of a stream transformation. This includes to **provide a data stream** to the transducer, and to **deal with the transformed stream** that they get from the transducer.

They are:
- the `from where` and the `to where`,
- the `how do I take that input` and the `what should I do with that output`.

And since they are the one which are calling the transducer, they are also in charge of deciding the `when do I do this to that`.

### Transducer users from the standard library

Note: In the following paragraph, the parameter `xform` refers to a transducer.

- [(into to xform from)](http://clojuredocs.org/clojure.core/into): When you want the output stream's elements to be all stored in a collection, which can be a vector, a list, a map, a sorted map, or any other type of other collection. I love that function because it appends to the collection [in a very efficient way](https://github.com/clojure/clojure/blob/clojure-1.9.0/src/clj/clojure/core.clj#L6820).

- [(sequence xform coll)](http://clojuredocs.org/clojure.core/sequence): When you want the stream to be processed "only when needed". Useful if you think that the sequence consumer may not need all the output elements, or if don't want all of it to be processed immediately. That's useful for throttling the transformation w.r.t. the CPU workload, or if you need to send the output stream over a slower channel like the network and you don't want to buffer the whole output stream in advance.

- [(eduction xform* coll)](http://clojuredocs.org/clojure.core/eduction): ... it's complicated, see the docs for a precise explanation. Rarely needed by the average programmer. It returns a non-lazy sequence which is evaluated using the transducers each time it is being used by a [`reduce`](http://clojuredocs.org/clojure.core/reduce) function.

- [(transduce xform f init coll)](http://clojuredocs.org/clojure.core/transduce): When you want to use the [`reduce`](http://clojuredocs.org/clojure.core/reduce) operation but you also want to perform a stream transformation on the data to be reduced.

### Transducer users from the `clojure.core.async` library

- [(chan buf-or-n xform)](http://clojuredocs.org/clojure.core.async/chan): Creates a channel where the data stream on the reader side will be the one of the writer side transformed by the transducer.

- [(pipeline n to xf from)](http://clojuredocs.org/clojure.core.async/pipeline): "Takes elements from the from channel and supplies them to the to channel, subject to the transducer xf, with parallelism n [...]". To be used only with stateless transducers. Has the [`pipeline-blocking`](http://clojuredocs.org/clojure.core.async/pipeline-blocking) variant.

- [(transduce xform f init ch)](http://clojuredocs.org/clojure.core.async/transduce): A variant of [`clojure.core/transduce`](http://clojuredocs.org/clojure.core/transduce) where the data stream comes from a channel instead of a collection.

### Other uses of transducers

There may be other functions that use transducers, adapted to different contexts. If you know some, please let me know by leaving a comment and I will add them if they are relevant to the audience.

# Let's implement some!

Good news, transducers are now your friends, you can call them when you want. But if you want to be a good friend, there are some rules to follow.

### The output function

Transducers are technically functions which transform a reducing function into another reducing function.

If you don't know what that mean, you can also see them as a functions that transform a kind of output function into another one, except that this output function is taking a kind of accumulator value as its first parameter.

Let's play with this idea a bit and let's implement a function that provides a stream of data from a collection to a transducer and outputs the transformed stream to the standard output, element by element.

```clojure
; Notes:
; - Don't do this at home, this is still a toy function.
; - The transducer is often called 'xf' and the reducer function 'rf'.
(defn print-duce [transducer coll]
  (let [reduction-fn #(print (str %2 \space))
        process (transducer reduction-fn)]
    (loop [c coll]
      (when (seq c)
        (do
          (process nil (first c))
          (recur (rest c)))))))

(print-duce (map inc) (list 3 4 5))
; Output:
; 4 5 6

; But this does not work:
(print-duce (partition-all 2) (list 3 4 5))
; Output:
; [3 4]
;
; the [5] is missing!
```

### {0 1 2}-Arity

![You said Arrietty?](/img/xf-tuto/ariety.jpg)

If you read the parts 1 to 4 of this blog serie, you will know that transducers are expected to be called with different arities.

0-Arity: **Just ignore it**, don't call it unless you know what you are doing (and let me know why in the comments or [in that StackOverflow question](https://stackoverflow.com/questions/48848876/transducers-init-not-called)).

2-Arity: Call it each time you want **to feed one more data element to the transducer**.

1-Arity: Call it once you to inform that there are no inputs available anymore. The transducer (even a stateless one) **may flush a few more data** to your output function.

Now let's improve our `print-ducer` a bit.

```clojure
; Note: Still a toy function.
(defn print-duce [xf coll]
  (let [rf (fn ([])
               ([result] (print (str \newline "--EOS")))
               ([result input] (print (str input \space))))
        process (xf rf)]
    (loop [c coll]
      (if (seq c)
        (do
          (process nil (first c)) ; 2-arity 'process'
          (recur (rest c)))
        (process nil)))))         ; 1-arity 'flush'

(print-duce (map inc) (list 3 4 5))
; Output:
; 4 5 6
; --EOS

; Now this works:
(print-duce (partition-all 2) (list 3 4 5))
; Output:
; [3 4] [5]
; --EOS
```

### Enough is enough (early termination)

Your transducer has its say on when to stop the stream. It will return a `reduced` value when it thinks that there should not be any other element. You need to pay attention to it and not ask too much.

```clojure
(defn print-duce [xf coll]
  (let [rf (fn ([])
               ([result] (print (str \newline "--EOS")))
               ([result input] (print (str input \space))))
        process (xf rf)]
    (loop [c coll]
      (if (seq c)
        (let [result (process nil (first c))]
          (if (reduced? result)
            (process (deref result)) ; unwraps it and stop processing the stream
            (recur (rest c))))       ; continue processing the stream
        (process nil)))))
```

Note that I used `deref` in order to unwrap the reduced value while I was using `unreduced` in the part 4. Those two functions are different, `deref` really does unwrap the reduced value, and [`unreduced`](http://clojuredocs.org/clojure.core/unreduced) is unwrapping a value **only if** it is reduced.

Source code:

```clojure
(defn unreduced [x]
  (if (reduced? x) (deref x) x))
```

### Reduce function as a parameter

What if we want to make our function more general and accept a "1-and-2-arity" reducing function as a parameter?

```clojure
(defn multi-duce [xf rf init coll]
  (let [process (xf rf)]
    (loop [acc init
           c coll]
      (if (seq c)
        (let [result (process acc (first c))]
          (if (reduced? result)
            (process (deref result))
            (recur result (rest c))))
        (process acc)))))

(multi-duce (map inc) conj [] (list 3 4 5))
; => [4 5 6]

(multi-duce (partition-all 2) conj [] (list 3 4 5))
; => [[3 4] [5]]
```

### Easy made simple (by my cat)

![This is not my cat](/img/xf-tuto/dilbert.jpg)

My cat just told me that I am an idiot as there is a shorter and more efficient way to implement the function above.

I gave the keyboard to him and here is what he shown me:

```clojure
(defn cat-duce [xf rf init coll]
  (let [process (xf rf)
        result (reduce process init coll)]
      (process result)))

(cat-duce (map inc) conj [] (list 3 4 5))
; => [4 5 6]

(cat-duce (partition-all 2) conj [] (list 3 4 5))
; => [[3 4] [5]]
```

Indeed, it seems to work. Thank you cat!

### The `transduce` function

The cat-duce function is very close to the implementation of the legendary [`transduce`](http://clojuredocs.org/clojure.core/transduce) function. It has the same signature and [almost the same implementation](https://github.com/clojure/clojure/blob/clojure-1.9.0/src/clj/clojure/core.clj#L6800), the difference being the added support for collections which want to be noticed when they are being reduced (like the result of the [`eduction`](http://clojuredocs.org/clojure.core/eduction) function).

# What's next

Congratulations !!!

By now you should probably know anything you need to use the transducers in your own custom way. You may still need to exercise your skills a little bit - **Practice makes perfect**.

<iframe width="755" height="566" src="https://www.youtube.com/embed/04854XqcfCY" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>


I am going to publish some transducer-related exercises on May 30th 2018 to be used as a support for a [transducer workshop in Taipei](https://clojure.tw/weekly/2018-05-20/). The link will be added here soon after.

Stay tuned!

**UPDATE**: I created a Github project with some [transducer exercises and their solutions](https://github.com/green-coder/transducer-exercises), feel free to give it a try and let me know if it helped.
