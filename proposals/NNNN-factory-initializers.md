# Indirect Initializers

* Proposal: SE-NNNN
* Author: [Riley Testut](http://twitter.com/rileytestut), [Gor Gyolchanyan](https://github.com/technogen-gg)
* Status: **Awaiting review**
* Review Manager: TBD

## Introduction

We propose adding an additional type of initializer to the Swift language, an `indirect` initializer. An `indirect` initializer will provide a way to return an instance that is not the exact type implied by the declaration context. When declared in a class, an instance of the class or one of its subclasses will be returned. When declared in a protocol extension, an instance of a conforming type will be returned.

Swift-evolution threads: [[Proposal] Factory Initializers](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20151214/003192.html), [[Proposal] Uniform Initialization Syntax](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170605/037275.html)

## Motivation

The "factory" pattern is common in many languages, including Objective-C. Essentially, instead of initializing a type directly, a method is called that returns an instance of the appropriate type determined by the input parameters. Functionally this works well, but ultimately it forces the client of the API to remember to call the factory method instead, rather than the type's initializer. This might seem like a minor gripe, but given that we want Swift to be as approachable as possible to new developers, we think we can do better in this regard.

## Proposed solution

Rather than have a separate factory method, we propose we build the factory pattern right into Swift, by way of specialized “indirect initializers”. Indirect initializers are able to directly return a value whose type is upperbounded by `Self`.

Additionally, indirect initializers would be able to be implemented via protocol extensions. This would allow developers to expose a protocol, and provide a way to instantiate a “default” type for the protocol, without having to also declare the default type as public. This is similar to instantiating an anonymous class conforming to a particular interface in Java.

## Detailed Design

1. The existing `indirect` keyword used for declaring indirect enums will be able to be applied to initializers, designating them indirect.
2. Compared to current initializers, indirect initializers must _return_ a value, not delegate calls to another initializer.
3. The returned value's type must be upperbounded by `Self`. The specifics of this depend on whether the indirect initializer is a member of a class, value type, or protocol extension:
    * Class: Returned value must be an instance of the class declaring the indirect initializer, or a subclass.
    * Value-Type: Returned value's type must be the same as the type declaring the indirect initializer.
    * Protocol Extension: Returned value's type must conform to the protocol being extended.
4. Indirect initializers would _not_ be able to be overridden by subclasses, just like convenience initializers.
5. Indirect initializers _must_ have a different method signature than the type's other initializers. This way, it is possible to call any other initializer on Self without ambiguity, allowing us to return an instance of Self in addition to any subclasses, such as here:

```swift
public class Base {

    private init(privateInfo: InformationToSwitchOn) {}

    public indirect init(info: InformationToSwitchOn) {
        if … {
            return Base(privateInfo: type) // Returns instance of type Self
        }
        else {
            return SpecificBase(privateInfo: type) // Returns instance of subclass of Self
        }
    }
}

private class SpecificBase : Base {}
```

## Examples

__Class__

```swift
class AbstractBase {

    private init(privateType: InformationToSwitchOn) {}
    
    indirect init(type: InformationToSwitchOn) {
        if … {
            return ConcreteImplementationOne(privateType: type)
        }
        else {
            return ConcreteImplementationTwo(privateType: type)
        }
    }
}

private class ConcreteImplementationOne : AbstractBase {}
private class ConcreteImplementationTwo : AbstractBase {}
```

__Protocol Extension__

```swift
protocol MyProtocol {}

extension MyProtocol {
    indirect init() {
        return ConformingStruct()
    }
}

private struct ConformingStruct: MyProtocol {}
```

__Class Cluster/Abstract Classes__  
This was the reasoning behind the original proposal, and I still think it would be a very valid use case. The public superclass would declare all the public methods, and could delegate off the specific implementations to the private subclasses. Alternatively, this method could be used as an easy way to handle backwards-compatibility: rather than litter the code with branches depending on the OS version, simply return the OS-appropriate subclass from the indirect initializer. Very useful.

__Initializing Storyboard-backed View Controller__  
This is more specific to Apple Frameworks, but having indirect initializers could definitely help here. Currently, view controllers associated with a storyboard must be initialized from the client through a factory method on the storyboard instance (storyboard.instantiateViewControllerWithIdentifier()). This works when the entire flow of the app is storyboard based, but when a single storyboard is used to configure a one-off view controller, having to initialize through the storyboard is essentially use of private implementation details; it shouldn’t matter whether the VC was designed in code or storyboards, ultimately a single initializer should “do the right thing” (just as it does when using XIBs directly). An indirect initializer for a View Controller subclass could handle the loading of the storyboard and returning the appropriate view controller.

## Source Compatibility

This proposal will have no impact on existing code. This will only add an additional way to instantiate types.

## Effect on ABI stability

This proposal could be implemented without affecting the ABI, however the existence of `indirect init` can provide static typing guarantees about non-indirect `init`, opening up optimization opportunities on the ABI level.

## Effect on API resilience

This proposal introduces changes to the initialization model, which even for public APIs does not change the syntax or behavior of existing code in any way, so any changes to the public API as a result of utilizing the features of this proposal are completely voluntary, so no Swift version checking is necessary.

## Alternatives considered

* Keep the Swift initialization pattern as-is. The Swift initialization pattern can already be somewhat complex (due in no part to Objective-C’s influence), and adding another rule to it can potentially confuse newcomers even more. That being said, there is currently no way to accomplish certain tasks, such as a class cluster pattern, in pure Swift without this addition.

* Limit factory initializers to classes/value types. This would work, but I believe there are some genuine benefits to allowing factory initializers on protocols as well. Because they simply return values, I don’t see a reason to *not* include them for protocols, in order to keep the initialization patterns as uniform as possible.

## Comments from Swift-Evolution

__Philippe Hausler <phausler@apple.com>__  
I can definitely attest that in implementing Foundation we could have much more idiomatic swift and much more similar behavior to the way Foundation on Darwin actually works if we had factory initializers. 

__Brent Royal-Gordon <brent@architechies.com>__  
A `protocol init` in a protocol extension creates an initializer which is *not* applied to types conforming to the protocol. Instead, it is actually an initializer on the protocol itself. `self` is the protocol metatype, not an instance of anything. The provided implementation should `return` an instance conforming to (and implicitly casted to) the protocol. Just like any other initializer, a `protocol init` can be failable or throwing.

Unlike other initializers, Swift usually won’t be able to tell at compile time which concrete type will be returned by a protocol init(), reducing opportunities to statically bind methods and perform other optimization tricks. Frankly, though, that’s just the cost of doing business. If you want to select a type dynamically, you’re going to lose the ability to aggressively optimize calls to the resulting instance.
