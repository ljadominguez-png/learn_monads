# Lab 1: The Safe Numeric Input (Hello World of Monads)

In this 1-hour exercise, we will build a minimal Flutter app that parses a user's text input into an integer and calculates its square.

Instead of writing a standard try-catch block inside your UI widget or checking for null, you will build your very own, zero-dependency Result<S, E> monad from scratch in plain Dart, connect it to a text field, and observe how it prevents invalid state transitions.

Step 1: Create the Monad (lib/result.dart)
Inside your Flutter project, create a new file named lib/result.dart.

1. **Define the Sealed Parent Class:**
Create the generic base container `Result<S, E>` where `S` represents the success value type and `E` represents the error type. Mark it with the `sealed` modifier so the compiler restricts all direct subclasses to this file, and give it a `const` constructor for immutability.

```dart
sealed class Result<S, E> {
  const Result();
  // Methods will be added here
}

```


2. **Declare the Concrete Subclasses:**
Define the two terminal variants: `Success<S, E>` holding `S value` and `Failure<S, E>` holding `E error`. Marking them `final` prevents any further inheritance downstream.

```dart
final class Success<S, E> extends Result<S, E> {
  final S value;
  const Success(this.value);
}

final class Failure<S, E> extends Result<S, E> {
  final E error;
  const Failure(this.error);
}

```


3. **Implement Functor Transformation with map:**
Inside `Result<S, E>`, implement `map<R>` using an exhaustive `switch (this)` expression. If the instance is `Success`, extract the `value`, pass it through `transform(value)`, and wrap it in a new `Success<R, E>`. If it is `Failure`, propagate `Failure(error)` unchanged.

```dart
Result<R, E> map<R>(R Function(S value) transform) {
  return switch (this) {
    Success(:final value) => Success(transform(value)),
    Failure(:final error) => Failure(error),
  };
}

```


4. **Implement Monadic Chaining with flatMap:**
Add `flatMap<R>` to chain operations that can themselves fail. Use another exhaustive `switch (this)` pattern. When handling `Success`, execute `nextStep(value)` directly (which already produces a `Result<R, E>`), avoiding nested results like `Result<Result<R, E>, E>`. If it is `Failure`, pass the error forward.

```dart
Result<R, E> flatMap<R>(Result<R, E> Function(S value) nextStep) {
  return switch (this) {
    Success(:final value) => nextStep(value),
    Failure(:final error) => Failure(error),
  };
}

```
