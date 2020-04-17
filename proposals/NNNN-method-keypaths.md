# Allow Key Paths to Refer to Unapplied Methods

* Proposal: [SE-NNNN](NNNN-method-keypaths.md)
* Authors: [Paul Cantrell](https://github.com/pcantrell), [Jeremy Saklad](https://github.com/Saklad5), [Filip Sakel](https://github.com/filip-sakel)
* Review Manager: TBD
* Status: **Awaiting implementation**

## Introduction

The [original key path proposal](https://github.com/apple/swift-evolution/blob/master/proposals/0161-key-paths.md)
expressly limited key paths to be able to reference only properties and subscripts:

```swift
\Person.name                  // KeyPath<Person, String>
\Person.pets[0]               // KeyPath<Person, Pet>
```

This proposal adds the ability for key paths to reference instance methods, optionally specifying argument names:

```swift
\Person.sing                  // KeyPath<Person, () -> Sound>
\Person.sing(melody:lyrics:)  // KeyPath<Person, (Melody, String) -> Sound>
```

Note that these key paths do not provide argument values; they reference _unapplied_ methods, and the value they give is
a function, not the the value that results from calling the method.

Adding this capability not only removes an inconsistency in Swift, but solves pratical problems involving map/filter
operations, proxying with key path member lookup, and passing weak method references that do not retain their receiver.

Swift-evolution thread: [Why can’t key paths refer to instance methods?](https://forums.swift.org/t/why-can-t-key-paths-refer-to-instance-methods/35315)

## Motivation

**Key paths** provide the ability to opaquely refer to type members without binding to a specific value of that type.
For example, `\String.count` refers to “the `count` property of `String`s in general,” without referencing or being
attached to any _particular_ `String` value.

A simple symmetry illustrates the nature of key paths: `foo.___.___` (an expression that contains a member access) is
equivalent to `foo[keyPath: \.___.___]` (the member access split out as a key path, then recombined). For example:

```swift
let exampleURL = URL(string: "https://swift.org/contributing")!

// (extra whitespace below to show parallel structure)

exampleURL           .host                   // Single property…
exampleURL[keyPath: \.host]                  // ✅ …works as a key path!

exampleURL           .absoluteString.count   // Chain of properties…
exampleURL[keyPath: \.absoluteString.count]  // ✅ …works as a key path!

exampleURL           .pathComponents[1]      // Subscript…
exampleURL[keyPath: \.pathComponents[1]]     // ✅ …works as a key path!

exampleURL           .scheme?.capitalized    // Optional chain…
exampleURL[keyPath: \.scheme?.capitalized]   // ✅ …works as a key path!
```

However, this symmetry breaks down when we reference a method:

```swift
exampleURL           .deletingLastPathComponent   // gives us a function of type () -> URL
exampleURL[keyPath: \.deletingLastPathComponent]  // ❌ error: key path cannot refer to instance method

exampleURL           .encode(to:)                 // gives us a function of type (Encoder) -> ()
exampleURL[keyPath: \.encode(to:)]                // ❌ error: key path cannot refer to instance method
```

This asymmetry is not pleasing. But is it a problem? Swift does have the very similar feature of **unbound method
references**:

```swift
let action = String.reversed  // unbound and unapplied: (String) -> () -> ReversedCollection<String>
action("foobar")              // bound but unapplied: () -> ReversedCollection<String>
action("foobar")()            // bound and applied: "raboof"
```

This construct is quite similar to a key path: it encapsulates a _member_ of a type in a form that lets us apply it
later to various _values_ of that type. For example, we can use it to rewrite the two ❌ failing examples above:

```swift
// Instead of \URL.deletingLastPathComponent…

let action0 = URL.deletingLastPathComponent  // (URL) -> () -> URL
action0(exampleURL)                          // equivalent to exampleURL.deletingLastPathComponent

// Instead of \URL.encode(to:)…

let action1 = URL.encode(to:)  // (URL) -> (Encoder) -> Void
action1(exampleURL)            // equivalent to exampleURL.encode(to:)
```

However, relying on unbound method references to fill in the gap in key paths leaves us lacking in several situations:

- **Chains that mix properties and methods**: While key paths currently cannot reference methods, unbound member
    references can _only_ reference methods.  For example, while `URL.deletingLastPathComponent` works, `URL.path` does
    not compile because `path` is a property. This means that there is no construct in Swift short of a fully
    spelled-out closure that can reference a chain that includes both properties _and_ methods.

    For example, the key path `\URL.absoluteString.reversed` will not compile because `reversed` is a method, and the
    unbound member reference `URL.absoluteString.reversed` will not compile because `absoluteString` is a property.
    This precludes some conveniences and idiomatic patterns the language would otherwise allow.

- **Abstractions over both properties and methods**: [SE-0252](https://github.com/apple/swift-evolution/blob/master/proposals/0252-keypath-dynamic-member-lookup.md)
    brings us tantalizingly close to being able to write type-safe dynamic proxies in Swift. However, because they rely
    on key paths, such proxies can currently only handle properties and not methods. While this proposal alone would not
    make fully robust proxies possible in Swift, it would extend the proxying that is possible.

- **Idioms that rely on contextual type**: When the contextual type is either `KeyPath<Root,_>` or `(Root) -> _`, we can
    omit `Root` from a key path literal. For example, given `var names: [String]`, instead of writing
    `names.map(\String.count)`, we can simply write `names.map(\.count)`.

    Unbound method references do not have this feature. Using `.someMember` in a context that could accept an unbound
    method will cause Swift to look for `.someMember` on the corresponding _function_ type — and since function types
    have no members, this will fail. The magic of key path is that, given `\.someMember`, contextual inference searches
    for `someMember` on the `Root` type of the key path instead of on `KeyPath` itself.

    There are some idioms where unbound methods could be useful, but are slightly too awkward without this particular
    contextual type convenience. For example, as we will demonstrate below, method key paths could help mitigate the
    [oft](https://forums.swift.org/t/pitch-weak-method-storage-modifiers-aka-weak-references/12161)-[discussed](https://forums.swift.org/t/pitch-introduction-of-weak-unowned-closures/6095)
    problem of weakly bound method references.


## Proposed solution

Allow key paths to reference methods, in addition to the properties and subscripts they currently support:

```swift
struct Foo {
    func bar() { }

    func baz(zonk: Int, blotz: String) -> Int { … }
}

\Foo.bar               // KeyPath<Foo, () -> Void>
\Foo.baz(zonk:blotz:)  // KeyPath<Foo, (Int,String) -> Int>
```

Note that applying the key path does not _call_ the method; rather, it creates an unapplied method reference — a closure
waiting for arguments.

This fixes the ❌ inconsistencies above:

```swift
exampleURL           .deletingLastPathComponent   // Unapplied method…
exampleURL[keyPath: \.deletingLastPathComponent]  // ✅ …works as a key path!

exampleURL           .encode(to:)                 // Unapplied method with named arguments…
exampleURL[keyPath: \.encode(to:)]                // ✅ …works as a key path!
```

Adding this feature opens up possibilties both pleasant and promising:

### Chains that mix properties and methods

We now have a way of referencing a member chain that includes both properties _and_ methods; the key path
`\URL.absoluteString.reversed` now compiles.

Why is such a construct useful? Here is an example from [real-world
code](https://github.com/bustoutsolutions/siesta/blob/3b5b83447a83bb70237537a93ecca18a6fa256b3/Source/Siesta/Pipeline/PipelineProcessing.swift#L164):

```swift
stagesAndEntries.prefix(upTo: index)
    .compactMap { $0.cacheEntry?.remove })  // gather removal actions to perform later
```

With this proposal, we could simplify the call above to:

```swift
stagesAndEntries.prefix(upTo: index)
    .compactMap(\.cacheEntry?.remove)  // gather removal actions to perform later
```

(Thanks, [SE-0249](https://github.com/apple/swift-evolution/blob/master/proposals/0249-key-path-literal-function-expressions.md)!)

### Abstractions over properties and methods

**TODO**: key path member lookup example

### Idioms that rely on contextual type

**TODO**: Siesta key path-as-observer example


## Detailed design

In general, for a type `T`and a method `m`, if and only if `T.m` evaluates to an unbound method of type
`(T) -> (A) -> B`, then:

- `\T.m` will evaluate to a key path of type `KeyPath<T, (A) -> B>`, and
- given `var x: T`, the expression `x[keyPath: \.m]` will be equivalent to the expression `x.m`.

Note that this phrasing aims to extend the current behavior of key paths according to existing precedent set by unbound
methods, and does not seek to make any novel design decisions or solve additional problems beyond that. In particular,
just as with unbound methods:

- You may provide argument names to disambiguate methods with the same name, and may omit argument names if the method
    reference is unambiguous without them:

    ```swift
    \Array<String>.reversed            // OK (reversed() has no args)
    \Array<String>.joined(separator:)  // OK
    \Array<String>.joined              // Also OK (no ambiguity)

    \Array<Int>.prefix          // compile error: ambiguous
    \Array<Int>.prefix(upTo:)   // OK
    \Array<Int>.prefix(while:)  // OK
    ```

- You may disambiguate methods overloaded by type using a contextual type:

    ```swift
    struct S {
        func f(arg: Int) { … }
        func f(arg: String) { … }
    }

    \S.f                               // compile error: ambiguous
    \S.f(arg:)                         // compile error: still ambiguous
    \S.f as KeyPath<S, (Int) -> Void>  // OK
    ```

- Method key paths cannot reference methods requiring a generic type or provided by a conditional conformance except when
    the reference is unambiguous:

    ```swift
    \Array.joined            // OK because joined is on an extension constrained to Element == String
    \Array.contains          // compile error: ambiguous
    \Array<String>.contains  // OK
    ```

- The root type of a method reference cannot be a protocol with Self or associated type requirements:

    ```swift
    \Sequence.min  // compile error

    // Use a generic type instead:
    func foo<T: Sequence>(…) {
        \T.min  // OK
    }
    ```

- In contrast to subscripts, whose arguments may be embedded in key paths, argument values / partial method application
    are _not_ allowed in this version of the proposal:

    ```swift
    \Array.joined(separator: "\n")  // syntax error
    ```

- Method key paths may not reference mutating methods:

    ```swift
    \Array.append  // compile error (how would we reference the mutated collection?)
    ```

- Method key paths may not reference type methods (i.e. static or class methods) or operators:

    ```swift
    \DispatchQueue.global(qos:)  // No
    \Array.+                     // Also no
    ```

    This one deserves a little explanation. The expression `DispatchQueue.global(qos:)` _does_ work in Swift today;
    however, it evaluates to a _bound but unapplied_ method of type `(DispatchQoS.QoSClass) -> DispatchQueue`. There is
    no way in Swift today to specify an _unbound_ class function reference that means “the class member `global(qos:)`
    of some _arbitrary subclass_ of `DispatchQueue`.” Such an unbound would require a caller to first specify which
    subclass of `DispatchQueue` it wants — remember this is a class method, so subclasses can override it! — and then
    specify the QoS parameter; it would have the type `(DispatchQueue.Type) -> (DispatchQoS.QoSClass) -> DispatchQueue`.

    Because the parallel unbound method construct does not exist in Swift today, key paths would not support it either
    (yet) under this proposal.

Again, **all of the above abilities and limitations** follow from our constraint that method key paths mirror existing
behavior; none of the above is a new design decision unique to this proposal.

These last three limitations — no partial application of arguments, no mutating methods, no type methods — are both ripe
for proposals of their own, but pose design questions best served by separate discusions (see Future Directions below),
and are **out of scope for this proposal**.

Any future proposals on topics such as these should consider both unbound methods and key paths, and attempt to maintain
coherence between the two as much as reasonably possible.

---

The one capability that method key paths method references possess that unbound methods do not is they may appear at the
end of a longer chain involving properties, subscripts, and optional chaining / unwrapping:

```swift
URL.baseURL?.pathComponents[1].reversed   // compile error: no chains allowed here, only one member deep
\URL.baseURL?.pathComponents[1].reversed  // OK!
```

Note that because (1) methods in key paths return functions and (2) functions themselves have neither members nor
subscripts, methods can only appear as the last element in a key path. This could change if a future proposal for
example allowed embedding of method arguments in a key path, or somehow allowed function types to conform to protocols.

---

Since unbound methods are never properties, and key paths are currently never methods, this proposal introduces a
conflict when a method and a property have the same name, e.g. `Array.first` and `Array.first(where:)`. In this case, a
bare reference with no argument names always refers to the property, and argument names are necessary to refer to the
method:

```swift
\[Int].first          // keypath to property
\[Int].first(where:)  // keypath to method
```

This parallels the existing behavior of bound but unapplied methods, which face the same problem and solve it in the
same way:

```swift
var a: [Int]

a.first          // property
a.first(where:)  // unapplied method
```

This rule guarantees that it is always possible to reference both the property and the methods via key paths, since it
is a compile error to declare a no-args method with the same name as a property:

```swift
struct S {
    var x: Int
    func x(_: Int) { … }   // allowed
    func x() { … }         // compiler error
}
```


## Source compatibility

This is a pure additive proposal: it only concerns key paths that end with method references, which currently do not
compile at all. In the case where a funciton and a property have the same same, any existing key path that compiles
today will continue to refer to the property.


## Effect on ABI stability

**TODO**: I don't think this affects the ABI, since the generated types (`KeyPath<Base,m->m>`) are representible in
Swift’s current type system even if the key path syntax doesn't support them. Are there ABI implications beyond that?
I’m not even sure what ABI impacts look like.


## Effect on API resilience

APIs might choose to expose or require key path types with function values, e.g. `KeyPath<String, () -> Int>`, as a
result of this proposal. However, these types are already legal in Swift; this proposal simply provides a new way to
construct such values.


## Future directions

**TODO**: discuss how mutating methods might play out so we know we're not painting ourselves into a corner

**TODO**: discuss named args in proxy example


## Alternatives considered

**TODO**: what are alternatives? maybe expanding unbound methods to properties?
