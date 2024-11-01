---
layout: post
title: "Exploring Javascript Generative Testing"
---

I've used generative testing on and off over the years, and want to explore it as a way to build more effective abstractions. I have been using Javascript as the primary language in my personal projects more recently, so to begin this research I want to better understand what's available in the current landscape.

First, though, let me talk about what I'm looking to explore. A lot of people talk about how unit testing is a reasonable approximation for proving the correctness of a program, but what's always sat uncomfortably with me is how easy it is to overlook corner cases. Generative testing offers the potential to more comprehensively cover a piece of code, and I wish to examine how it can be used to more comprehensively cover the contract it fulfills.

# [TestCheck.js](https://github.com/leebyron/testcheck-js)

A basic property is defined like this:

```javascript
const { check, gen, property } = require('testcheck');

const result = check(
  property(
    gen.int,
    x => x - x === 0
  )
)
```

I suppose I could write something to look at that result, but that can be a little clunky when I'm looking to use this frequently. It does offer test framework integration, but because it was last updated 6 years ago it doesn't support [Vitest](https://vitest.dev/) which appears to be the current front runner.

I can still wrap it in a test with an expectation, though, so still pretty easy to play around with:

```javascript
test("x - x", () => {
    const result = check(
        property(
            gen.int,
            x => x - x === 0
        )
    );
    expect(result.result).toBeTruthy();
});
```

Here's the basic call signature, stripped of specific examples:

```javascript
check(
    property(
        gen.<type>,
        <some function returning a boolean>
    ),
    <options>
)
```

There are a ton of different generators, enough that most conceivable cases would be covered. It's also possible to construct new generators by composing the ones provided, which gives a reasonable amount of extensibility.

# [fast-check](https://github.com/dubzzz/fast-check)

Fast Check appears to be more actively maintained and feature rich, but the documentation isn't as well organized. The basic usage is pretty much identical:

```javascript
test("x - x == 0", () => {
    fc.assert(
        fc.property(
            fc.integer(),
            (x) => x - x == 0
        )
    )
})
```

Because it's actively maintained, though, it supports modern testing libraries and so I didn't have to examine the result to get the test to fail.

This library also seems to provide many generators which can be extended to check most things I could imagine.

# Conclusion

The basic footprint of all JS generative testing libraries is the same, although there only appears to be one library actively maintained. Even though it's not too cumbersome, the footprint in both cases is a bit clunky and leads to a lot of extra code to transition between a simple test and a property based test.
