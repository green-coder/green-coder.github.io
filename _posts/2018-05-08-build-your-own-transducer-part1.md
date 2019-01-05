---
layout: clojure-post
title: Build Your Own Transducer and Impress Your Cat - Part 1
description: Brief introduction to what transducers are and how to use them.
date: 2018-05-08
tags: [Clojure, Transducer, Beginners]
categories: Clojure
---

This post is a part of a serie:

1. Introduction to transducers (this post)
2. [Anatomy of a transducer](2018-05-10-build-your-own-transducer-part2.md)
3. [Stateful transducers](2018-05-12-build-your-own-transducer-part3.md)
4. [Early termination in transducers](2018-05-13-build-your-own-transducer-part4.md)
5. [Functions which are using transducers](2018-05-21-build-your-own-transducer-part5.md)
6. [Transducer exercises and solutions](https://github.com/green-coder/transducer-exercises)

---

This article is a brief introduction to what transducers are and how to use them.

# Hello Transduced World

In simple words:

> A transducer is the equivalent of an operation on a stream of data elements.

That man is a kind of transducer, it takes data elements from its input and may send some on its output:
![A kind of transducer](/img/xf-tuto/human_resource_machine01.png "from the game 'Human Resource Machine'")

You can see it as a function which is called exactly once for each element of the input stream, and which can spit any number of elements to its output stream.

There is a special emphasis on the fact that:
- the transducer is not taking anything from the input stream by itself, it is not his job or responsibility. It gets called (by something else) with each data element.
- the transducer only knows that it has to call a designated (and obscure) function with each of it output elements and has to return its result. It has no idea where the data goes.

🤓 Note: _If you want to play at being yourself a transducer in a funny fictional world with an intuitive and simple programming syntax, [give this cool game a try](https://en.wikipedia.org/wiki/Human_Resource_Machine)._

# Vincent, please focus

Oh okay, right.

Here is the simplest transducer:

```clojure
(map identity)
```

It accepts some data elements as input and gives the exact same data elements to its output. Except that at the moment it has no friends, nobody gave him data and it was not provided an output. A transducer alone is like a stomach without food (hmm .. let's not talk too much about the output in that case, but notice that it still needs a place to go).

Let's use that transducer properly. Here the `into` function gives it some data elements from a list and collects its output elements into a vector.

```eval-clojure
(into ["b" "a"] (map identity) (list "n" "a" "n" "a"))
```

Another example with a transducer that actually does something with its input, it transforms each character into its preceding character in the alphabet.

```eval-clojure
(into ["b" "a"]
      (map #(char (dec (.charCodeAt % 0))))
      (list "u" "n" "b" "o"))
```

Notice that you can use any function `f` which takes 1 value as parameter and returns 1 value by using the form `(map f)`. Simple and easy.

Now let's see some built-in transducers which do not have the same number of input and output elements.

This one outputs 1 or none output element for each input element:

```eval-clojure
(into []
      (filter #(<= (.charCodeAt "a" 0) (.charCodeAt % 0) (.charCodeAt "f" 0)))
      (list "c" "r" "a" "f" "e" "b" "h" "a" "b" "l" "e"))
```

This one gives 1 or more output elements for each input element.

```eval-clojure
(into []
      (mapcat #(if (<= 0 % 9)
                 (list % %)
                 (list %)))
      (list 10 5 16 7 13))
```

_Notes: The function provided to `mapcat` returns a collection of arbitrary size and `mapcat` returns a transducer which outputs an arbitrary number of elements for each input element._

# Will it blend?

Multiple transducers can be piped together to give birth to more complex data processing pipelines. This is done via the `comp` function and the result of the composition is a transducer. The data elements flow one after another through the pipeline, no buffer is used for storing intermediary results between each of its steps - that's a streaming process.

```eval-clojure
(into []
      (comp
        (map inc)                 ; first step
        (filter odd?)             ; second step
        (mapcat #(if (<= 0 % 9)   ; third step
                   (list % %)
                   (list %))))
      (list 8 9 10 11 12))
```

# The built-in transducers

Clojure has a few transducers which are very popular because they are handy.

[TODO: insert a link to a post that my future self will write about it]


# What's next

[In the next post](2018-05-10-build-your-own-transducer-part2.md), I talk about the anatomy of a transducer and how to program your own.
