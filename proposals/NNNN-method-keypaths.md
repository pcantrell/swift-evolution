# Allow key paths to reference unapplied instance methods

* Proposal: [SE-NNNN](NNNN-method-keypaths.md)
* Authors: [Paul Cantrell](https://github.com/pcantrell), [Filip Sakel](https://github.com/filip-sakel), [Jeremy Saklad](https://github.com/Saklad5)
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
a function, not the the value that results from calling the method. (See [Future Directions](#future-directions).)

Adding this capability not only removes an inconsistency in Swift, but also solves pratical problems involving map/filter
operations, proxying with key path member lookup, and passing weak method references that do not retain their receiver.

Swift-evolution thread: [Why can’t key paths refer to instance methods?](https://forums.swift.org/t/why-can-t-key-paths-refer-to-instance-methods/35315)

## Motivation

**Key paths** provide the ability to opaquely refer to type members without binding to a specific value of that type.
For example, `\String.count` refers to “the `count` property of `String`s in general,” without referencing or being
attached to any _particular_ `String` value.

A simple symmetry illustrates the nature of key paths: `foo.___.___` (an expression that contains a member access) is in
many cases equivalent to `foo[keyPath: \.___.___]` (the member access split out as a key path, then recombined). For
example:

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
    omit `Root` from a key path literal. For example:

    ```swift
    var names: [String]

    names.map(\String.count)  // OK
    names.map(\.count)        // Also OK: compiler infers String, like magic!
    ```

    Unbound method references do not have this feature. Using `.someMember` in a context that could accept an unbound
    method will cause Swift to look for `.someMember` on the corresponding _function_ type — and since function types
    have no members, this will fail:

    ```swift
    names.map(String.reversed)  // OK
    names.map(.reversed)        // error: type '(String) throws -> _' has no member 'reversed'
    ```

    The magic of the key path is that, given `\.someMember`, Swift searches for `someMember` on the key path’s `Root`
    instead of on the type `KeyPath` itself.

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

Adding this feature opens up possibilties both pleasant and promising.

### Use cases

#### Chains that mix properties and methods

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

#### Abstractions over properties and methods

Having a single abstraction that can reference both properties and methods opens new possibilities. For example, this
proposal combined with dynamic member lookup is powerful. It allows us to create boilerplate-free wrapper types that
support method calls:

```swift
@dynamicMemberLookup
struct StringWrapper {
    var string: String

    subscript<Value>(dynamicMember member: KeyPath<String, Value>) -> Value {
        string[keyPath: member]  // possibly wrapped with additional processing
    }
}

let wrapper: StringWrapper = …

wrapper.dropFirst()  // Now valid code; returns value of type `Substring`.
```

Note that, per [SE-0111](https://github.com/apple/swift-evolution/blob/master/proposals/0111-remove-arg-label-type-significance.md),
Swift values of function type do not allow argument labels. This means the code above strips argument labels from the
wrapped methods:

```swift
var hasher: Hasher = …
let wrapper = StringWrapper(string: "myString")
"swift".hash(into: &hasher)  // String method has label
wrapper.hash(&hasher)        // Wrapper method has no label
```

This proposal is thus not a complete solution for dynamic proxies. It does, however, advance the state of the art in the
language.

Having a uniform way to refer to methods and properties can also help produce more natural and readable code. For
example, with a few additional `map` methods:

```swift
extension Array {
    func map<U>(_ transform: @escaping (Element) -> () -> U) -> () -> [U] { … }
    func map<T,U>(_ transform: @escaping (Element) -> (T) -> U) -> (T) -> [U] { … }
    // etc
}
```

…we can write fluent map/filter chains that traverse both properties and methods:


```swift
let myArray: [URL] = …

myArray
    .map(\.resolvingSymlinksInPath)()
    .filter(\.isFileURL)
    .map(\.path)
    .map(\.dropFirst)(3)
```

#### Idioms that rely on contextual type

A common mistake [Siesta](https://bustoutsolutions.github.io/siesta/) users make is the following:

```swift
resource.addObserver(owner: self, action: self.handleChange)
```

Because of Siesta’s memory management policy (details not important here), this code creates a retain cycle. The correct
way to write this snippet is to use a closure that weakly captures `self`:

```swift
resource.addObserver(owner: self) { [weak self] resource, event in
    self?.handleChange(resource, event)
}
```

The verbosity of this construction means developers almost never use it in practice. Siesta designs around it by
encouraging developers to instead provide an object that implements a `ResourceObserver` protocol, and it provides a
shortcut overload to encourage that pattern. This design decision is a direct result of the current awkwardness of
forming weakly bound method references.

Others have also noted the need for weakly bound method references, and over the years there have been
[multiple](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170213/032478.html)
[different](https://forums.swift.org/t/pitch-weak-method-storage-modifiers-aka-weak-references/12161)
[pitches](https://forums.swift.org/t/pitch-introduction-of-weak-unowned-closures/6095) for simplifying the syntax for
creating and/or enforcing them.

A currently available alternative that sidesteps this problem is to provide an overload of the `addObserver` method that
accepts an _unbound_ method reference, and assumes the receiver should be the object passed as `owner`:

```swift
resource.addObserver(owner: self, action: CurrentViewController.handleChange)  // addObserver weakly references owner
```

This approach works, and has the advantage that is allows `addObserver` to control how it retains the method receiver.
However, it is still prohibitively verbose — especially when the wrong but tantalizing `self.handleChange` sits close at
hand. As [Chris Lattner noted](https://forums.swift.org/t/proposal-universal-dynamic-dispatch-for-method-calls/237/62)
about Swift’s decision to use `let` instead of `const`, developers will be prone to reach for the wrong thing and then
rationalize it if it takes fewer keystrokes!

(Curiously, `Self.handleChange` currently does not work in this context — and if it did, the visual difference between
`self.handleChange` (wrong) and `Self.handleChange` (correct) would be just a bit too insidiously close for comfort.)

Method key paths provide a more elegant alternative:

```swift
resource.addObserver(owner: self, action: \.handleChange)
```

Key paths let us omit the type name from `action` where unbound methods do not. How? The signature of this `addObserver`
overload is as follows:

```swift
func addObserver<T: AnyObject>(owner: T, action: KeyPath<T, ResourceObserverClosure>)
```

When a key path literal omits the type name, Swift uses the key path’s _root_ type as the contextual type instead of the
key path itself. In this case, that means the key path literal `\.handleChange` looks for a `handleChange` method on
`T`, and thus needs no type prefix. This feature — unique to key paths — is what makes this idiom work.

This idiom could become commonplace for APIs that require a method reference to be weakly bound. (It could even
generalize to a callable `WeakUnappliedMethod` type, given some additional language work to support more finely typed
`@dynamicCallable` and tuple splatting.)


## Detailed design

In general, for a type `T`and a method `m`, if and only if `T.m` evaluates to an unbound method of type
`(T) -> (A) -> B`, then:

- `\T.m` will evaluate to a read-only key path of type `KeyPath<T, (A) -> B>`, and
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

- Key paths cannot reference methods rooted in a generic type or provided by a conditional conformance unless the root
    type is unambiguous:

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
    are _not_ allowed in this version of the proposal (see [Future Directions](#future-directions)):

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
    subclass of `DispatchQueue` it wants — remember this is a class method, so subclasses can override it! — and then
    specify the QoS parameter; it would have the type `(DispatchQueue.Type) -> (DispatchQoS.QoSClass) -> DispatchQueue`.

    Because the parallel unbound method construct does not exist in Swift today, key paths would not support it either
    (yet) under this proposal.

Again, **all of the above abilities and limitations** follow from our constraint that method key paths mirror existing
behavior; none of the above is a new design decision unique to this proposal.

These last three limitations in the list above — no partial application of arguments, no mutating methods, no type
methods — are all ripe for proposals of their own, but pose design questions best served by separate discusions (see
[Future Directions](#future-directions)), and are **out of scope for this proposal**.

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
is a compile-time error to declare a no-args method with the same name as a property:

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

**TODO**: _I don't think this affects the ABI, since the generated types (`KeyPath<Base,(T)->U>`) are representible in
Swift’s current type system even if the key path syntax doesn't support them. Are there ABI implications beyond that?
I’m not even sure what ABI impacts look like. -PPC_


## Effect on API resilience

APIs might choose to expose or require key path types with function values, e.g. `KeyPath<String, () -> Int>`, as a
result of this proposal. However, these types are already legal in Swift; this proposal simply provides a new way to
construct such values.


## Future directions

Some popular feature ideas are out of this proposal’s scope. The discussion below explains why.

### Method calls in key paths

This proposal deals only with _unapplied_ methods. There is also a strong case to make for supporting _applied_
methods, i.e. allowing key paths to embed method calls.

For example, one can transform a `URL` to another `URL` both by resolving relative paths (via `absoluteURL`) and by
resolving symlinks (via `resolvingSymlinksInPath()`). Yet one of these is a property and the other is a method, so this
proposal only supports making a key path that returns the value of the former. It seems natural that key paths should
support both:

```swift
\URL.absoluteURL                // This works...
\URL.resolvingSymlinksInPath()  // ...so why shouldn’t this? But this proposal doesn’t cover it

\URL.resolvingSymlinksInPath    // With proposal above, this gives a () -> URL function, not a URL
```

What about methods that take arguments? Key paths already support embedded _subscript_ arguments:

```swift
\URL.pathComponents[1]          // 1 is embedded in the key path
\URL.pathComponents[1].count    // chain can continue after the subscript
\URL.pathComponents[i + j * 3]  // subscript can contain expression, eagerly evaluated
```

This suggests that key paths ought to be able to embed _method_ arguments in a similar fashion:

```swift
\URL.appendingPathComponent("extra")  // KeyPath<URL, URL>
```

This natural line of reasoning runs into trouble when we realize that key path subscripts can only embed `Hashable`
arguments. This is already a pain point for subscripts; for method calls, it would be untenable. The reasoning behind
the `Hashable` restriction and underlying implementation concerns would complicate this otherwise straightforward
proposal. We have thus chosen to leave method calls out of scope here.

### Partially applied methods

There has also been longstanding desire from the Swift community for _partial method application_, where a method
reference provides some but not all arguments and evaluates to a closure that accepts the remaining ones:

```swift
// ⚠️ hypothetical syntax for illustrative purposes, not a proposal ⚠️
url.appendingPathComponent(pathComponent: _, isDirectory: true)  // (String) -> URL
```

It would make sense for key paths to support this too:

```swift
\URL.appendingPathComponent(pathComponent: _, isDirectory: true) // KeyPath<URL, (String) -> URL>
```

This idea has surfaced on Swift Evolution
[as long ago as 2016](https://forums.swift.org/t/proposal-automating-partial-application-via-wildcards/1284)
and [as recently as April 2020](https://forums.swift.org/t/thoughts-regarding-the-potential-assignment-of-functions-to-labelled-identifiers/35471).
It even made an appearance in the venerable
[SE-0002](https://github.com/apple/swift-evolution/blob/master/proposals/0002-remove-currying.md#alternatives-considered).
However, it quickly becomes entangled in
[doubts about Swift’s method reference syntax](https://forums.swift.org/t/require-parameter-names-when-referencing-to-functions/27048)
that raise thorny source compatibility questions.

Whatever approach such a feature ultimately takes, the **syntax should be uniform across method references and key
paths**. And whatever syntax compatibility challenges a solution raises for key paths already exist for method
references. Thus while this document’s proposal to bringing key paths into alignment with method references does not
_solve_ the problem, neither does it _change_ it.

Given that, we think it best not to let this relatively simple proposal die on the rocks by opening the “partial method
application” can of worms.

### Mutating methods

This proposal does not support mutating methods, because it is not clear how a caller should access the result of the
mutation. It seems natural that this code should work:

```swift
var primes = [2, 3, 5]
primes[keyPath: \.append](7)
```

…but what, then, should this do?

```swift
var primes = [2, 3, 5]
let appender = primes[keyPath: \.append]
appender(7)  // Is primes now mutated? Does appender keep a hidden pointer to
             // a stack-allocated value? What if appender escapes local scope?
```

Key paths for mutating methods might induce an `inout` param for self:

```swift
let appender: (inout Array<Int>, Int) -> Void = \Array.append
appender(primes, 7)
```

…but that idea only makes sense when using key path literals as a convenience to form closures; it quickly breaks down
when using actual key path values as subscripts:

```swift
var mersennes = [0, 1, 3]
primes[keyPath: \.append](mersennes, 7)  // Which is mutated: mersennes or primes?
```

This line of reasoning suggests that mutating methods are best left for a follow-up proposal:

- If they are in fact limited to key path _literals_ converted to closures, then they are both outside the scope of this
  proposal and not in conflict with it.
- If we seek a broader solution, it would need to be consistent with unbound methods as well, and thus (as in the
  previous section) this proposal is only uniformly respecting an existing problem, not introducing a new one.

### Support for argument labels

Accepting this proposal will create pressure to solve the argument label problem in the
[wrapper type example above](#abstractions-over-properties-and-methods).

One possibility would be loosening SE-0111 to make argument labels optional:

```swift
struct Foo {
    func bar(a: A) { ... }
 }

struct FooWrapper { ... }
let wrappedFoo: FooWrapper = ...

wrappedFoo.bar(a: A())
// this would be equivalent to:
wrappedFoo.bar(a:)(A())
```

Alternatively, a
[recent pitch](https://forums.swift.org/t/include-argument-labels-in-identifiers/35367/23) suggested that argument
labels be added to the identifier instead of the function type. It might be possible to extend this to identifiers
dynamically resolved by key paths.

However, given that SE-0111 and its implications have generated copious discussion without resolution
([1](https://forums.swift.org/t/review-se-0111-remove-type-system-significance-of-function-argument-labels/3209),
[2](https://forums.swift.org/t/update-commentary-se-0111-remove-type-system-significance-of-function-argument-labels/3391),
[3](https://forums.swift.org/t/accepted-se-0111-remove-type-system-significance-of-function-argument-labels/3306/17),
[4](https://forums.swift.org/t/se-0111-related-question/3813),
[5](https://forums.swift.org/t/se-0111-and-curried-argument-labels-unintended-consequences/4179)),
trying to solve the argument label problem as part of this proposal would likely doom it — and its benefits — to
purgatory.


## Alternatives considered

### Focus on unbound methods instead of key paths

This proposal extends the domain of key paths. We could instead extend the domain of unbound methods:

```swift
String.count  // yields a (String) -> Int function
```

This however does not provide the [contextual type inference benefits of key paths](#idioms-that-rely-on-contextual-type),
departs from `@dynamicMemberLookup`’s current focus on key paths, and would create confusing error messages for
developers who confuse instance properties with type properties.

### Prohibit method key paths in dynamic member lookup

This would prevent wrappers from exposing label-stripped methods:

```swift
let wrapper: SomeStringWrapper = ...

aString.hash(into: &value)  // This is how the call is supposed to look, so...
wrapper.hash(&value)        // ❌ we could disallow this until language supports `into:` label
wrapper.dropFirst()         // ❌ which would also disallow this
```

The argument in favor of this is it would be best not to expose methods stripped of labels now if we expect the labels
to return in the future, since their re-addition _might_ be source-breaking.

However, dynamic member lookup of methods is a popular motivation for this proposal. The argument label problem already
exists for unbound methods, so any attempt to solve it could already lead to source-breaking changes even without this
proposal. Such an attempt could avoid (or handle) that breakage in a uniform way across unbound methods and key paths.
