---
layout: post
title: Build Your Own Transducer and Impress Your Cat - Part 3
description: Brief introduction to what transducers are and how to use them.
tags: Clojure, Transducer, Beginners
categories: Clojure
---

This post is a part of a serie:

1. [Introduction to transducers](/clojure/build-your-own-transducer-part1/)
2. [Anatomy of a transducer](/clojure/build-your-own-transducer-part2/)
3. Stateful transducers (this post)
4. [Early termination in transducers](/clojure/build-your-own-transducer-part4/)
5. [Functions which are using transducers](/clojure/build-your-own-transducer-part5/)
6. [Transducer exercises and solutions](https://github.com/green-coder/transducer-exercises)

---

# Stateful transducers

This article describes briefly how to implement a stateful transducer.

## When do we need them

We need them each time some output elements depend on multiple input elements.

Some information related to previous input values need to be stored until the transducer has enough of them to start outputting values.

The information stored could simply be the input values, or it could be a derived information (which usually takes less space in memory).

## Stringifier Transducer

Suppose that we have a stream of chars representing zero terminated strings and we want to have a stream of strings.

```clojure
(defn string-builder-transducer [separator]
  ; The local state *should not* be defined here.
  (fn [rf]
    ; The local state of the transducer comes here.
    (let [state (volatile! [])]
      (fn ([] (rf))
          ([result] (-> result
                        (rf (apply str @state))
                        (rf)))
          ([result input]
           (let [chars @state]
             (if (= separator input)
               (do (vreset! state [])
                   (rf result (apply str chars)))
               (do (vreset! state (conj chars input))
                   result))))))))

(into []
      (string-builder-transducer 0)
      (list \H \e \l \l \o 0 \C \l \o \j \u \r \e 0 \w \o \r \l \d \!))
; => ["Hello" "Clojure" "world!"]
```

Three notes:

* The position where the local state should be introduced matters. If you introduce it at the wrong place, some local state will linger when the transducer is used in multiple pipelines and cause bugs.
* `volatile!` and `vreset!` are used instead of atoms for efficiency.
* In this example, the transducer simply stores the encountered chars until the end of each chunk and then used them all at once to create a string. It does not have to always be this way (e.g. the next paragraph).

## Chunk Sum Transducer

Suppose that we have a stream of numbers separated by a special keyword `:|` and we want to compute the sum of each chunk of numbers.

```clojure
(defn chunk-sum-transducer [separator]
  (fn [rf]
    (let [state (volatile! 0)]
      (fn ([] (rf))
          ([result] (-> result
                        (rf @state)
                        (rf)))
          ([result input]
           (let [acc @state]
             (if (= separator input)
               (do (vreset! state 0)
                   (rf result acc))
               (do (vreset! state (+ acc input))
                   result))))))))

(into []
      (chunk-sum-transducer :|)
      (list 1 2 3 4 :| 42 :| :| 3 5))
; => [10 42 0 8]
```

Note that our transducer is efficient in terms of memory as it only keep a sum of numbers in its local state instead of all the elements of the current chunk.

## Packet Transducer

Another example a little more complex, this transducer groups messages together in packets of data so that the size of the packet does not exceed its maximum limit (if possible, otherwise it uses bigger packets).

Here we use a map containing multiple values as the local state.

```clojure
(defn packet-transducer [max-size]
  (fn [rf]
    (let [state (volatile! {:packet []
                            :size 0})]
      (fn ([] (rf))
          ([result]
           (let [{:keys [packet size]} @state]
             (cond-> result
                     (pos? size) (rf packet) ; Flush the local state to output.
                     :always (rf))))         ; Transmit the flush signal.
          ([result input]
           (let [{:keys [packet size]} @state
                 input-size (count input)
                 new-size (+ size input-size)]
             (if (<= new-size max-size)
               (do (vreset! state {:packet (conj packet input)
                                   :size new-size})
                   result)
               (do (vreset! state {:packet [input]
                                   :size input-size})
                   (cond-> result
                           (pos? size) (rf packet))))))))))

(into []
      (packet-transducer 5)
      (list [1 1] [2 2] [3 3 3] [4 4] [5] [6 6 6 6 6]))
; => [[[1 1] [2 2]] [[3 3 3] [4 4]] [[5]] [[6 6 6 6 6]]]
```

## What's next

[In the next part of this blog serie](/clojure/build-your-own-transducer-part4/), I talk about how to support early termination in a data pipeline by using `reduced` and `reduced?`.
