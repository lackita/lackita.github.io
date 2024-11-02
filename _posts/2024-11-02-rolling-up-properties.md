---
layout: post
title: "Rolling Up Properties"
---

Now that I have a basic understanding of how the Javascript property libraries work, it's time to shift my attention to examining my assumptions around the idea I'm attempting to explore. Generative testing promises to provide much more comprehensive of the properties which it can verify, and what I'd like to examine is if those attributes can be automatically composed when used in higher level abstractions.

I'll use some simple mathematical properties as a way to explain what I mean. Consider, for example, a property true of the `>` relationship:

```javascript
property(
    integer(), integer(), integer(),
    (a, b, c) => (a > b && b > c) ? a > c : true,
)
```

Here's a simple function that could be built on top of that relationship:

```javascript
function positive(x) {
    return x > 0;
}
```

We of course know that, if `j > k` and `k` is positive, then `j` must also be positive. What I'm interesting in determining is if, because I defined the transitive property as a generative test, such a property could be automatically determined.

Before I embark on trying to pry apart the mechanics of how to figure something like that out, though, I need to validate this makes logical sense. To accomplish this, we somehow must infer that `c == 0` in the original property; perhaps easier said than done, but assuming it is determined leads to the following property:

```javascript
property(
    integer(), integer(),
    (a, b) => (a > b && b > 0) ? a > 0 : true,
)
```

Then we can transform this again by replacing with the definition of `positive`:

```javascript
property(
    integer(), integer(),
    (a, b) => (a > b && positive(b)) ? positive(a) : true,
)
```

These transformations are far from trivial in most languages, but shows it's at least possible to make logical transformations from base properties to more elaborate ones. This kind of meta-analysis might be more than Javascript is capable of in a practical way, but now I at least know it's logically possible to reach such conclusions.
