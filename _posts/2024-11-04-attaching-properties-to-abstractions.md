---
layout: post
title: "Attaching Properties to Abstractions"
---

A challenge around suggesting higher level properties is how to associate them to the abstractions they represent. For example, if I write a function like this:

```javascript
function equals(x, y) {
    return x == y;
}
```

How would I know that this property is related to that function:

```javascript
property(
    integer(), integer(),
    (x, y) => equals(x, y) == equals(y, x),
)
```

Obviously it's calling the function, but how do we figure out if the call is incidental or an attribute of the function itself. Static analysis is an option, but that can be challenging to build and sequencing the steps would be somewhat complex to orchestrate.

Other languages provide a way to annotate functions and classes, but in the Javascript world the only conversations I can find are around documentation. Javascript is a pretty dynamic language, though, so maybe there's something clever we can do there. Here's the property, definition and test reorganized a bit; it doesn't get us where we want to go, but helps organize things a little differently.

```javascript
const equals_property = property(
    integer(), integer(),
    (x, y) => equals(x, y) == equals(y, x),
)
function equals(x, y) {
    return x == y;
}

test("symmetric", () => {
    assert(
        equals_property
    )
})
```

One possible idea is to use first order functions to wrap function creation in a way that attaches a provide property:

```javascript
function def(property, func) {
    func.property = property;
    return func;
}

const equals = def(
    property(
        integer(), integer(),
        (x, y) => equals(x, y) == equals(y, x),
    ),
    (x, y) => x == y,
)

test("symmetric", () => {
    assert(
        equals.property
    )
})
```

So then it does seem possible to attach a property directly to the associated function. It might get a little complicated when we begin looking at defining objects like this, but this is enough to demonstrate that it's possible to establish a direct link between a piece of logic and a property.
