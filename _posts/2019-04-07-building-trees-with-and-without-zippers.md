---
layout: clojure-post
title: Building trees with and without zippers
description: This article shows how to harvest the power of zippers to build trees out of sequences of data.
date: 2019-04-07
tags: [Clojure, Data-structure, Zipper]
categories: Clojure
---

In his [blog post and video](https://lambdaisland.com/blog/2018-11-26-art-tree-shaping-clojure-zip)
from november 2018, Arne Brasseur explains the idea behind the functional zippers
in the `clojure.zip` namespace.
He describes how to use them to navigate existing tree-shaped data structures
and how to modify them.

There is something that he did not mention explicitly, it is that they are **an excellent choice** when you don't
have a tree but need to build one.

# From sequence to tree

As an example, suppose that you have the following sequence of data:

```clojure
(def elements [{:type :todo-list
                :name "Things I should do today"}
               {:type :text
                :value "Buy some bread"}
               {:type :url-list
                :title "Read blogs"}
               {:type :url
                :value "https://vincent.404.taipei"}
               {:type :url
                :value "https://lambdaisland.com/blog"}
               {:type :text
                :value "Look for some interesting Clojure blogs."}
               {:type :url-list-end}
               {:type :foo
                :value :bar}
               {:type :todo-list-end}
               {:type :life
                :description "Take the time to relax"}])
```

Now, suppose that what you really want is a tree structure like that one:

```clojure
[{:type :todo-list
  :name "Things I should do today"
  :children [{:type :text
              :value "Buy some bread"}
             {:type :url-list
              :title "Read blogs"
              :children [{:type :url
                          :value "https://vincent.404.taipei"}
                         {:type :url
                          :value "https://lambdaisland.com/blog"}
                         {:type :comment
                          :value "Look for some interesting Clojure blogs."}]}
             {:type :foo
              :value :bar}]}
 {:type :life
  :description "Take the time to relax"}]
```

In order to do the transformation, you would intuitively need to:
- find the beginning and the end of each block in the sequence,
- build a tree node for each block,
- have the elements of each block's sub-sequence placed in that block as children,
- look for the other blocks to process.

## The recursive algorithm

![Recursion](/img/zipper/recursion.png "From XKCD")

Supposing that we already have the full sequence of data available when we want
to process it, we could recursively build our tree by iterating on the sequence
in this way:

```clojure
(defn build-tree [data-sequence]
  (letfn [;; Returns [built-children unconsumed-sequence]
          (append-children [children [element & next-sequence :as data-sequence]]
            ;; End of the tree construction.
            (if (nil? data-sequence)
              [children nil]

              (case (:type element)
                ;; If we need to open a new block.
                (:todo-list :url-list)
                (let [[sub-node-children unconsumed-sequence]
                      (append-children [] next-sequence)]
                  (append-children (conj children (assoc element :children sub-node-children))
                                   unconsumed-sequence))

                ;; If we need to close the current block.
                (:todo-list-end :url-list-end)
                [children next-sequence]

                ;; If we want to add a leaf node to the tree.
                (append-children (conj children element) next-sequence))))]

    (first (append-children [] data-sequence))))

;; Transforms the sequence into a tree.
(build-tree elements)
```

That implementation is hard to understand simply because
it does too many things at the same time:
- Build a tree, nodes and children.
- Keeps track of where we are in the sequence, **in and out** of recursive calls.
- Iterate on the sequence and handle termination.

It also has limitations:
- Messy code flow, hard to read and maintain with **3 different recursion calls**
  combined in a complicated way.
- Does not check the match between elements like `:url-list` and `:url-list-end`,
  would need an additional parameter for that (sigh!).
- Not an incremental algorithm.

That type of recursive function takes a lot of time to write and debug.
It is not good for its writer, and it is not good for the co-workers who
have to read it at some point later in the future.

We certainly can - and therefore we **should** - do better.

## Improvements

Important observation:

> The tree is being built at the same time than the data-sequence is being read.

Instead of organizing the tree's construction around the shape of
the tree, we could organize it around the shape of the sequence. After all,
thinking linearly is way more easy than thinking on a tree.

The idiomatic way to process sequences in Clojure is to use the
[reduce](http://clojuredocs.org/clojure.core/reduce) operation.
For building our tree, it would look like this:

```clojure
(reduce (fn [tree data-element]
          ;; Add the element to the existing tree, return the updated-tree.
          (some-magic tree data-element))
        empty-tree
        data-sequence)
```

Once we have an algorithm that updates the tree in a reducing process,
we know that our algorithm is incremental, as the reduce function is
an incremental process.

```clojure
(reduce f :init [1 2 3])

;; The above can be rewritten as:
(-> :init
    (f 1)  ; process one data
    (f 2)  ; process another data
    (f 3)) ; and so on, and so on
```

## The incremental algorithm

We need to replace the `some-magic` function above with some real code.
Since we don't have magic, let's compensate using analytical thinking.

### Tracking the node to edit

Adding an element to a tree is easy, but knowing where to add it is not trivial.
The recursive algorithm knew where to add them because it implicitly kept track
of the node being currently edited.

To emulate this, we need to **keep track of that node**.

### "I am your father"

A shortcoming of the recursive algorithm was that there was no easy way to access
the other existing nodes of the tree being built.

Ideally, it would be nice for the **whole tree to be accessible** at any time,
specially **the nodes close to the node been edited**.

### The zipper solution

![](/img/zipper/zipper.jpg "Zipper, photo by https://www.flickr.com/photos/southpaw2305/")

Zippers have both of the features above:
- They keep track of a tree structure and a current location in that tree.
- They provide an access to the whole tree at any time of the editing,
  from the node being tracked.

Here is how the whole function looks like using a zipper:

```clojure
(require '[clojure.zip :as z])

(defn build-tree [data-sequence]
  (let [;; Build a custom zipper
        empty-tree-zipper
        (z/zipper (comp #{:root :todo-list :url-list} :type)  ; branch?
                  :children                                   ; children
                  (fn [node children]                         ; make-node
                    (assoc node :children (vec children)))
                  {:type :root})                              ; root

        ;; The incremental update function
        update-zipper
        (fn [zipper element]
          (case (:type element)
            (:todo-list :url-list) (-> zipper
                                       (z/append-child element)
                                       (z/down)
                                       (z/rightmost))
            (:todo-list-end :url-list-end) (z/up zipper)
            (z/append-child zipper element)))]

    ;; Process the whole sequence
    (-> (reduce update-zipper
                empty-tree-zipper
                data-sequence)

        ;; Return the tree data structure from the zipper
        z/root)))

(build-tree elements)
```

Noteworthy:

- The `update-zipper` function is the most important part,
it only take 8 lines of code and takes about 15 seconds to read and understand it.
- Modifying the code to check for matching open/close elements is trivial and is
left to the reader as an exercise.
- The function that manipulates the tree only do that, the function that defines
how to update node's children is defined elsewhere.

# What's next

In the next article, I will talk about a real life problem where a tree needed
to be built, and which was solved elegantly by using a custom zipper.
