---
layout: post
title: "Understanding Abstractions"
---

A skill I often see underdeveloped in developers is layering simple
abstractions. Maintaining this discipline is crucial in dodging code
quagmires, which can be deadly when bugs occur. This has always been
important, but because genies don't understand this concept
maintaining this quality is becoming central to the value we bring.

To illustrate this concept, I'm going to provide an example from a
recent project I've been working on. On a Go board, adjacent points in
same state are considered connected:

![A 5x5 annotated Go board in this configuration: NNNNN/BBBN(4)N/WWB(1)NN/NWW(3)B(2)B/N(5)NWWB](/images/go-groups.png)

So the groups marked 1, 2 and 3 are each distinct in this regard. I'm
also going to refer to 4 and 5 as connected groups, this is useful for
the logic I'm developing even though Go players don't refer to them in
this way.

I want to develop a cache of these groups, which can be accessed
efficiently using a javascript `Map`. As new stones are added and
removed from the board, this will need to be updated accordingly.

Smaller is Better
=================

A point on the board is comprised of x and y coordinates, which feels
natural to represent as `[x, y]`. It's tempting to also the state of
that position, but this is the first lesson: the more sprawling the
abstraction the harder it is to reason about it.

To illustrate this point, let me share with you a simple utility
function I use in several places in this project:

```javascript
function neighbors([x, y]: Point): Point[] {
  return [[1, 0], [-1, 0], [0, 1], [0, -1]]
    .map(([ox, oy]) => [x + ox, y + oy])
}
```

Because it doesn't need to be aware of the state of the board, it can
remain self contained and more generic. If it needs the state, how do
we figure out what to include there? One option would be to put some
sort of dummy value:

```javascript
function neighbors([x, y, _]: Point): Point[] {
  return [[1, 0], [-1, 0], [0, 1], [0, -1]]
    .map(([ox, oy]) => [x + ox, y + oy, " "])
}
```

This would be functional, but adds unnecessary weight to the
abstraction. Now, whenever you use the results of this function, you
have to keep in mind that the state of that point should be
disregarded. Eventually, you'll accidentally use that state and have
a confusing bug to track down when it leaks into other parts of the
code.

The other option is to provide enough context to provide meaningful
state:

```javascript
function neighbors([x, y]: Point, board: Board): Point[] {
  return [[1, 0], [-1, 0], [0, 1], [0, -1]]
    .map(([ox, oy]) => {
      const nx = x + ox
      const ny = x + oy
      return [nx, ny, board.get(nx, ny)]
    })
}
```

Again, this would be functional, but introduces different
problems. To start, when writing the tests for this function, a
`Board` has to be constructed to provide this state. Next, knowledge
of the state is helpful in some situations but can become a burden in
others, which I'll come back to when I discuss sets of points.

Grouping Reduces Complexity
===========================

Discussing the entire `Board` class is beyond the scope of this
article, but notice that because `Point` doesn't include state we can
have a signature like this: `get(point: Point): State | undefined`. In the
previous example, because state was part of the type, we had to call a
version that looked like this: `get(x: number, y: number): State | undefined`.

To understand the consequence of this, consider the situation where
you only want neighbors that are valid board positions:

```javascript
class Board {
  ...
  neighbors(point: Point): Point[] {
    return neighbors(point).filter((n) => this.get(n))
  }
  ...
}
```

Notice that this function doesn't need to understand anything about
how `Point` is structured. If it was aware of state, then to call
`get` we would have to break into the individual x and y
coordinates. Moreover, it ceases to make sense to decompose the work
into two simple functions.

```javascript
class Board {
  ...
  neighbors([x, y, _]: Point): Point[] {
    return [[1, 0], [-1, 0], [0, 1], [0, -1]]
      .map(([ox, oy]) => board.get(x + ox, y + oy))
      .filter(n => n)
  }
  ...
}
```

What you're witnessing here is how
[balls of mud](https://www.laputan.org/mud/) begin to form. The wrong
abstraction was chosen for `Point`, so logic has begun accumulating
instead of being decomposed. The former version was compact and
bootstrapped from a more general concept, whereas the latter must know
about everything to accomplish its goal.

Also notice how much harder it is to understand the latter version,
instead of taking it in at a glance you have to understand the goal of
the low level transformations. Genies might be able to determine that
rapidly, but it takes more tokens to do so and over an entire codebase
this can lead to slower and more expensive development costs.

Unnecessary Weight
==================

Because Javascript's `Set` implementation would not work as expected
(the abstraction is
[leaky](https://www.joelonsoftware.com/2002/11/11/the-law-of-leaky-abstractions/)),
I had to develop `PointSet` to handle these. The implementation is
rather boring, so I won't share it here, but it's a class I use all
over the place.

Part of what makes it so useful is that it conforms to the
[Single Responsibility Principle](https://en.wikipedia.org/wiki/Single-responsibility_principle).
It has one job and does it well, and I rarely think about how it
accomplishes its goal. Keeping `Point` to x/y coordinates is the
secret sauce of why I never have to think about it, to understand why
let's explore what would happen if it included state.

If we include state as part of what makes a point unique, it could be
put in a state where the same coordinates can be in two states
simultaneously. Even if you never care about the state, it's still
vulnerable to this kind of problem:

```javascript
const set = new PointSet()
set.add([0, 0, "B"])
set.add([0, 0, "W"])
set.remove([0, 0, "W"])
```

When `remove` is called, it's reasonable to expect that `(0, 0)` is no
longer in the set. If you iterate over the members, though, you'll
find that a phantom `[0, 0, "B"]` is still hanging around.

To address this, we could consider ignoring the state when it comes to
uniqueness. As long as you're the only one to ever interact with the
class, it might be feasible to keep that detail in your head. Now that
we have genies, though, this is virtually never true. In my
experience, this manifests as a hallucination assuming that uniqueness
extends to the state. As a result, every interaction with the class
grows defensive logic, guarding against the possibility that an
existing point exists with a different state.

Both of these options are sub-optimal, and the consequences add weight
to interacting with the class. It's a detail that needs to be tracked,
adding to either your mental or the genie's context. If you try to
shed that weight, it results in subtle errors or superfluous code,
both of which simply shift the weight into debugging time or the
codebase.

Levels of Abstraction
=====================

Now we can finally discuss the group cache, which will use a two
tiered `Map` to provide rapid lookup.

```javascript
export class GroupCache {
  groups: Groups
  constructor(starting_groups?: Groups) {
    if (starting_groups) this.groups = starting_groups
    else this.groups = new Map()
  }

  at(point: Point): PointSet | undefined {
    return this.get(point)!.points
  }
  ...
}
```

To add members to this cache, an `update` function is defined:

```javascript
update(point: Point, state: State) {
  this.remove_from_existing_group(point, state)
  this.set(point, {
    state,
    points: this.add_to_new_group(point, state),
  })
}
```

What's important to understand here is that all of this logic operates
at the same level. No part of `at` or `update` looks inside of
`point`, they delegate to `get` and `set` to operate on those:

```javascript
private get([x, y]: Point): PointGroup | undefined {
  return this.groups.get(x)?.get(y)
}

private set([x, y]: Point, point_group: PointGroup) {
  if (!this.groups.has(x)) this.groups.set(x, new Map())
  this.groups.get(x)!.set(y, point_group)
}
```

What's interesting is that, although these operations aren't complex,
they have some nuance protecting against common errors. For example,
`set` first ensures that the map exists for `x` before attempting to
call `set` on it. None of that matters from the perspective of
`update`, though, it can focus on what it means to update instead of
having to muck about with data structure massaging.

Genies
======

As I've alluded to a few times, this kind of thinking and design
skills are precisely what's needed when working with coding
assistants. My first attempt at creating this cache was done by Codex,
which tightly bound the cache to capturing rules from the
game. Sometimes that's good enough, but in this case it fell apart
when I started tracking captures for scoring purposes.

When you take care to form clean abstractions, though, these
assistants can really sing. When I was working on
`remove_from_existing_group`, I needed to break apart a single
`PointSet` into subsets that were connected. This logic is a bit
hairy, but because the goal had such a clean definition, I was able to
generate that functionality rapidly.

The takeaway isn't that genies are awful or wonderful, it's to better
understand what they are good and bad at. This article explores that
boundary to help you better understand how to navigate it.
