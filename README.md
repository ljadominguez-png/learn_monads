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

Step 2: Write the Business Logic (lib/calculator.dart)
Create lib/calculator.dart. Here we write two small rules:

Parse a string into an integer safely (no uncaught FormatException).

Reject numbers that are zero or negative.

1. **Parse and Validate the Raw String:**
Implement `tryParseInteger(String rawInput)` to convert user input into a safe `Result<int, String>`. Check for empty strings after trimming, attempt `int.tryParse()`, and return a descriptive `Failure` if parsing fails or a `Success(parsed)` if successful.

```dart
Result<int, String> tryParseInteger(String rawInput) {
  final trimmed = rawInput.trim();
  if (trimmed.isEmpty) {
    return const Failure('Input cannot be empty');
  }

  final parsed = int.tryParse(trimmed);
  if (parsed == null) {
    return Failure('"$trimmed" is not a valid integer');
  }

  return Success(parsed);
}

```


2. **Define Domain Business Rules as Fallible Operations:**
Write `ensurePositive(int number)` to enforce constraints on the parsed value. Because domain validation can also fail, it returns `Result<int, String>` rather than a raw `bool` or throwing an exception.

```dart
Result<int, String> ensurePositive(int number) {
  if (number <= 0) {
    return const Failure('Number must be strictly greater than 0');
  }
  return Success(number);
}

```


3. **Chain the Fallible Step Using flatMap:**
Call `tryParseInteger(rawInput)` to kick off the pipeline, then chain `.flatMap(ensurePositive)`. Because `ensurePositive` returns another `Result`, `flatMap` flattens the hierarchy and halts execution immediately if parsing failed, bypassing the validation entirely.

```dart
// Partial pipeline:
tryParseInteger(rawInput).flatMap(ensurePositive)

```


4. **Transform the Final Success Value Using map:**
Chain `.map((n) => n * n)` onto the verified number. Since squaring an integer cannot fail, use `map` to apply a pure transformation without needing to wrap the output in a `Result` manually.

```dart
Result<int, String> computeSquare(String rawInput) {
  return tryParseInteger(rawInput)
      .flatMap(ensurePositive)
      .map((n) => n * n);
}

```