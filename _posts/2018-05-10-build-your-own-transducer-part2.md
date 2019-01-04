---
layout: post
title: Build Your Own Transducer and Impress Your Cat - Part 2
description: Brief introduction to what transducers are and how to use them.
date: 2018-05-10
tags: [Clojure, Transducer, Beginners]
categories: Clojure
---

This post is a part of a serie:

1. [Introduction to transducers](2018-05-08-build-your-own-transducer-part1.md)
2. Anatomy of a transducer (this post)
3. [Stateful transducers](2018-05-12-build-your-own-transducer-part3.md)
4. [Early termination in transducers](2018-05-13-build-your-own-transducer-part4.md)
5. [Functions which are using transducers](2018-05-21-build-your-own-transducer-part5.md)
6. [Transducer exercises and solutions](https://github.com/green-coder/transducer-exercises)

---

# Anatomy of a transducer

This article describes briefly the structure of a transducer and how to implement one.

## Simple one-one mapping

Let's suppose that we want to manually create a transducer that increments numbers. We could normally just use `(map inc)`, but for the training we will do it from scratch.

```clojure
(def inc-transducer
  (fn [rf]
    (fn ([] (rf))                                   ; 0-arity aka 'the useless'
        ([result] (rf result))                      ; 1-arity aka 'the flusher'
        ([result input] (rf result (inc input)))))) ; 2-arity aka 'the doer'

(into [] inc-transducer (list 4 5 6))
; => [5 6 7]

; idiomatic way:
; (into [] (map inc) (list 4 5 6))
```

The `rf` function is processing the output value of our transducer. It does ... 'something' to return a merged (or not) version of the `result` value with the processed data's value `(inc input)`, and our transducer needs to return that new result.

The `rf` function could be just another transducer which we composed with, or it could be a terminal reducing function (hence the name `rf`).

The `0-arity` function is (IHMO) a useless bogus convention as we cannot rely on it being called for sure by the functions which use transducers. Just transmit the call to the next transducer/rf, maybe it will do something with it in some specific context, who knows.

The `1-arity` function is called by the function who uses the transducer when there is no more data to be processed. That's where transducers can flush their data if they had some in a local state (more on this possibility later).

The `2-arity` is where the input data gets processed and passed to the next function in the pipeline.

## One-one mapping with parameters

Now let's suppose that instead of incrementing the numbers we want to add them a given value, then we need a transducer with a parameter.

```clojure
(defn add-transducer [n]
  (fn [rf]
    (fn ([] (rf))
        ([result] (rf result))
        ([result input] (rf result (+ input n))))))

(into [] (add-transducer 3) (list 4 5 6))
; => [7 8 9]

; idiomatic way:
; (into [] (map #(+ 3 %)) (list 4 5 6))
```

## No Rabbit transducer (One-Some)

We want a transducer that makes the rabbits disappear, to illustrate the case where the transducer may not provide a new output value.

```clojure
(defn magician-transducer [animal]
  (fn [rf]
    (fn ([] (rf))
        ([result] (rf result))
        ([result input]
          (if (= animal input)
              result             ; Just don't "merge" the input into the result.
              (rf result input))))))

(into [] (magician-transducer :rabbit) (list :dog :rabbit :lynel))
; => [:dog :lynel]

; idiomatic ways:
; (into [] (remove #(= :rabbit %)) (list :dog :rabbit :lynel))
; (into [] (filter #(not= :rabbit %)) (list :dog :rabbit :lynel))
```

No rabbit, no problem.

## More cats transducer (One-Two)

And what if we want more cats now? (more data output than input)

```clojure
(defn glitch-transducer [animal]
  (fn [rf]
    (fn ([] (rf))
        ([result] (rf result))
        ([result input]
         (if (= animal input)
             (-> result
                 (rf input)
                 (rf input)) ; Send the input twice to the output pipeline.
             (rf result input))))))

(into [] (glitch-transducer :cat) (list :dog :cat :lynel))
; => [:dog :cat :cat :lynel]

; idiomatic way:
; (into []
;       (mapcat #(if (= :cat %) (list % %) (list %)))
;       (list :dog :cat :lynel))
```

More cats. Neo would be happy.

## RLE decompression (One-Many)

Suppose that we want to send a serie of values in one go to the output but we can't do it as in the previous example because the number of repeats is not fixed or is too big and we are lazy and sane. We can use `reduce` to output the values one by one (now you can see why `rf` is called like that, it can be seen as a reducing function).

```clojure
(def rle-decoder-transducer
  (fn [rf]
    (fn ([] (rf))
        ([result] (rf result))
        ([result [count data]]
         (reduce rf result (repeat count data))))))

(into []
      rle-decoder-transducer
      (list [0 :a] [1 :b] [2 :c] [3 :d]))
; => [:b :c :c :d :d :d]

; idiomatic way:
; (into []
;       (mapcat (fn [[count data]] (repeat count data)))
;       (list [0 :a] [1 :b] [2 :c] [3 :d]))
```

## What's next

All the transducers shown above are **stateless**: Their behavior is fully described by their inputs and their initial immutable parameters.

[In the next part of this blog post](2018-05-12-build-your-own-transducer-part3.md), I cover the **stateful** transducers, those with a local mutable state.
