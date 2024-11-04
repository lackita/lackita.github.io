---
layout: post
title: "Expression Substitution"
---

I did a bit of hand waving around substituting expressions in my last post to prove out the idea of property inference, but these kinds of operations are anything but trivial. To examine this more thoroughly, let's take an extremely simple function:

```javascript
function equals(x, y) {
    return x == y;
}
```

Let's also take an extremely simple property related to this expression:

```javascript
property(
    integer(), integer(),
    (x, y) => equals(x, y) == equals(y, x),
)
```

Our goal is to identify some mechanism which can determine that such a property can be inferred in a function calling `equals`:

```javascript
function double_equals(x, y) {
    return equals(2*x, 2*y);
}
```

Building this example, though, I'm beginning to see a counterexample to the original premise. Of course, `double_equals` retains the symmetry of `equals`, but something like this wouldn't:

```javascript
function is_double(x, y) {
    return equals(x, 2*y);
}
```

Perhaps there's a special circumstance when the structure of both arguments are identical, but this feels like it's diving too heavily into the specifics of mathematics. It's also possible there's more advanced things to explore in this area, but I'm looking to tackle something less complex.

So instead of confidentally inferring all properties from lower level abstractions, I'm envisioning a system which can extrapolate candidates properties. It would propose symmentry for both `double_equals` and `is_double`, but leave it up to the user to determine its relevance. It also gives the user an opportunity to define a revised version of the property such as this:

```javascript
property(
    integer(), integer(),
    (x, y) => is_double(x, y) == is_double(y*2, x/2)
)
```

I didn't achieve what I set out to at the beginning of this article, diving into the practical mechanics of substitution, but I was able to identify a flaw in the original approach. The original idea might eventually be possible, but this revised approach proves to be a much more tractable problem.
