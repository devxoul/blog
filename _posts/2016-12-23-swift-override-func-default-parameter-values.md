---
layout: post
title: "Overriding methods with default parameter values in Swift"
description: "The way to override methods with extra parameter which have default values."
---

I was trying to override a method from the super class with the extra logging purpose parameters: `file`, `function` and `line`. So I wrote the code like below:

```swift
class Parent {
  func foo() {
    print("Parent Foo")
  }
}

class Child: Parent {
  override func foo(bar: String = "Hello") {
    print("Child Foo (bar=\(bar))")
  }
}
```

But I got an error:

> error: method does not override any method from its superclass

It was because the method signature of the `Parent`'s `foo()` and the `Child`'s `foo(bar:)` are different even if we could execute both methods without any arguments. So I removed the `override` keyword from the `foo()` method.

```swift
class Child: Parent {
  func foo(bar: String = "Hello") {
    print("Child Foo (bar=\(bar))")
  }
}
```

This time I was able to compile the code but I couldn't get the expected result. I expected the `Child`'s `foo()` to be executed but `Parent`'s was executed.

```swift
Child().foo() // Parent Foo
```

I tried a lot with many ways and came up with the solution: marking original method unavailable.

```swift
class Child: Parent {
  @available(*, unavailable)
  override func foo() {
    // compiler will not use this
  }
  
  func foo(bar: String = "Hello") {
    print("Child Foo (bar=\(bar))")
  }
}
```

Finally I got the expected result.

```swift
Child().foo() // Child Foo (bar=Hello)
```

I used this trick in subclassing `MoyaProvider` from [Moya](https://github.com/Moya/Moya) to log where the `request()` method is called.

```swift
class MyProvider<Target>: MoyaProvider<Target> where Target: TargetType {
  @available(*, unavailable)
  override func request(_ token: Target) -> Observable<Response> {
    return super.request(token)
  }

  func request(
    _ token: Target,
    file: StaticString = #file,
    function: StaticString = #function,
    line: UInt = #line
  ) -> Observable<Response> {
    print("\(function):\(line) - REQUEST: \(token.method) \(token.path)")
    super.request(token)
  }
}
```
