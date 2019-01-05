---
layout: clojure-post
title: Build Your Own Transducer and Impress Your Cat - Part 4
description: Brief introduction to what transducers are and how to use them.
date: 2018-05-13
tags: [Clojure, Transducer, Beginners]
categories: Clojure
---

This post is a part of a serie:

1. [Introduction to transducers](2018-05-08-build-your-own-transducer-part1.md)
2. [Anatomy of a transducer](2018-05-10-build-your-own-transducer-part2.md)
3. [Stateful transducers](2018-05-12-build-your-own-transducer-part3.md)
4. Early termination in transducers (this post)
5. [Functions which are using transducers](2018-05-21-build-your-own-transducer-part5.md)
6. [Transducer exercises and solutions](https://github.com/green-coder/transducer-exercises)

---

# Early termination and `reduced?` values in transducers

Suppose that you want to write a transducer that uses some of the elements in the input stream and then ignore anything after it. It would be a waste to have to iterate on the remaining values of the stream if we do not use them.

```eval-clojure
(defn tired-map-transducer [func energy tired-answer]
  (fn [rf]
    (let [state (volatile! energy)]
      (fn ([] (rf))
          ([result] (rf result))
          ([result input]
           (rf result
               (let [energy @state]
                 (if (pos? energy)
                   (do
                     (vreset! state (dec energy))
                     (func input))
                   tired-answer))))))))

(into []
      (tired-map-transducer inc 5 :whatever)
      (list 1 2 3 4 5 6 7 8 9 10))
```

We certainly want to find a way to let the function calling the transducers to stop calling them.

We could do that in multiple ways, like sending a special value `:enough` after the last data. But what if that special data `:enough` appears in some input data stream? Sending it to the output would be misleadingly interpreted as the transducer's message to early terminate.

The Clojure core team came up with something that is very unlikely to appear in the input data stream by luck (or we could call it a bug and blame the user instead :-) ), and established it as a convention for transducers to inform their calling function about a value being the last one their would like to return.

That convention is: `(reduced last-value)`

```eval-clojure
(defn responsible-map-transducer [func energy]
  (fn [rf]
    (let [state (volatile! energy)]
      (fn ([] (rf))
          ([result] (rf result))
          ([result input]
           (let [energy @state]
             (vreset! state (dec energy))
             (cond-> (rf result (func input))
               (<= energy 1) reduced)))))))

(into []
      (responsible-map-transducer inc 5)
      (list 1 2 3 4 5 6 7 8 9 10))
```

`reduced` is simply returning a wrapped value. A value can be tested by the calling function as being wrapped or not by using the predicate `reduced?`.

```eval-clojure
(reduced? 7)
```

```eval-clojure
(reduced? (reduced 7))
```

## Are we done, then?

There is one more thing to check before calling it done.

![Look both ways when crossing the railway.](/img/xf-tuto/train-hide-other-train.jpg)

Let's test our transducer a little more. Is it still going to work in those cases?

```clojure
(into []
      (comp
        (responsible-map-transducer inc 5)  ; Step 1
        (responsible-map-transducer inc 3)) ; Step 2
      (list 1 2 3 4 5 6 7 8 9 10))
; => [3 4 5]
; It works fine.

(into []
      (comp
        (responsible-map-transducer inc 3)  ; Step 1
        (responsible-map-transducer inc 5)) ; Step 2
      (list 1 2 3 4 5 6 7 8 9 10))
; => [3 4 5]
; It works fine.
```

It all works fine. So why did I ask? Because you have to know why it still works.

The implementation of `responsible-map-transducer` only took care of what it returns when it has no energy. Nothing special was done with the result of `(rf result ...)` in the normal case, it was simply returned as is. If it was a `reduced?` value, it means that the `rf` function wanted an early termination. Our transducer has to make sure that in this case the returned value is `reduced`.

It happens to be the case here because we simply returned what the `rf` function returned, so we are in luck. This may not always be the case, so always pay attention and watch in both ways:

* Handle the case where your transducer wants an early termination,
* Handle the case where `rf` wants an early termination.

## The devil is in the details

Did you think about testing that?

```eval-clojure
(into []
      (comp
        (responsible-map-transducer inc 3)  ; Step 1
        (responsible-map-transducer inc 3)) ; Step 2
      (list 1 2 3 4 5 6 7 8 9 10))
```

You didn't? Too bad because you should have: It triggers a bug and you get an extremely unclear error message now.

![Confused Baby](/img/xf-tuto/confused-baby.jpg)

Something happened when we returned a value which was wrapped at the same time by both transducers: `(reduced (reduced last-value))`

The function which is calling the transducers (i.e. `into`) is supporting early termination by expecting either a value or a `reduced` value, in which case it is getting the unwrapped value via `(unreduced reduced-value)`. But instead of getting the value, it still gets a reduced value.

```eval-clojure
(reduced? (unreduced (reduced (reduced 7))))
```

How to fix it? Either we ask the Clojure core team to change all the functions which are using transducers to support multi-wrapped values, or either we avoid wrapping the values more than once.

## "There is a function for that" --- Rich Hickey

We can use the function [`ensure-reduced`](http://clojuredocs.org/search?q=ensure-reduced) from the standard library. Its implementation is very simple:

```eval-clojure
(defn ensure-reduced [x]
  (if (reduced? x) x (reduced x)))
```

Now let's correct our transducer.

```eval-clojure
(defn correct-map-transducer [func energy]
  (fn [rf]
    (let [state (volatile! energy)]
      (fn ([] (rf))
          ([result] (rf result))
          ([result input]
           (let [energy @state]
             (vreset! state (dec energy))
             (cond-> (rf result (func input))
               (<= energy 1) ensure-reduced))))))) ; <- The difference is here

(into []
      (comp
        (correct-map-transducer inc 3)  ; Step 1
        (correct-map-transducer inc 3)) ; Step 2
      (list 1 2 3 4 5 6 7 8 9 10))

; Idiomatic way:
; (into []
;       (comp
;         (take 3)   ; Step 1
;         (map inc)  ; Step 2
;         (map inc)) ; Step 3
;       (range 1 11))

;
```

## What's next

You already know everything you need to implement your own transducers. I encourage you to try some ideas on your own. If you encounter blocking problems in your adventure, feel free to leave a comment and I will try to help you (when I have time).

[In the next part](2018-05-21-build-your-own-transducer-part5.md), I talk about functions like `into` which are **using** the transducers and how to write one.
