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

Step 3: Connect to the UI (lib/main.dart)

1. **Bootstrap the Application:**
Create the application entry point `main()` and wrap the root with `MaterialApp`. Configure Material 3 theming with a color seed and establish `MonadPlaygroundScreen` as the home destination.

```dart
void main() {
  runApp(const MonadApp());
}

class MonadApp extends StatelessWidget {
  const MonadApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      theme: ThemeData(useMaterial3: true, colorSchemeSeed: Colors.deepPurple),
      home: const MonadPlaygroundScreen(),
    );
  }
}

```


2. **Initialize State and Controller:**
Define the `StatefulWidget` and its state class. Store the user's raw text using a `TextEditingController` and declare the state variable directly as an optional monad: `Result<int, String>? _calculationResult`.

```dart
class MonadPlaygroundScreen extends StatefulWidget {
  const MonadPlaygroundScreen({super.key});

  @override
  State<MonadPlaygroundScreen> createState() => _MonadPlaygroundScreenState();
}

class _MonadPlaygroundScreenState extends State<MonadPlaygroundScreen> {
  final TextEditingController _controller = TextEditingController();
  Result<int, String>? _calculationResult;
  // ...
}

```


3. **Connect UI Events to the Pipeline:**
Create the `_onCalculate` trigger. When the user taps the run button, pass the raw string from `_controller.text` into `computeSquare()` and wrap the assignment inside `setState` to trigger a widget rebuild.

```dart
void _onCalculate() {
  setState(() {
    _calculationResult = computeSquare(_controller.text);
  });
}

```


4. **Construct the Input and Action Layout:**
Build a standard form layout containing the `TextField` to capture user input and an `ElevatedButton` bound to `_onCalculate`.

```dart
TextField(
  controller: _controller,
  decoration: const InputDecoration(
    labelText: 'Enter an integer',
    hintText: 'e.g. 5, -2, or abc',
    border: OutlineInputBorder(),
  ),
  keyboardType: TextInputType.text,
),
const SizedBox(height: 16),
ElevatedButton(
  onPressed: _onCalculate,
  child: const Text('Run Pipeline (Square)'),
),

```


5. **Exhaustively Pattern Match the Result into UI:**
Use a collection-if and Dart 3 switch expression inside the widget tree to unwrap the monad. The compiler guarantees that both `Success` and `Failure` are handled without missing edge cases, rendering customized cards based on the extracted values.

```dart
if (_calculationResult != null)
  switch (_calculationResult!) {
    Success(:final value) => Card(
        color: Colors.green.shade50,
        child: Padding(
          padding: const EdgeInsets.all(16.0),
          child: Text(
            'Success! Result is: $value',
            style: TextStyle(
              color: Colors.green.shade900,
              fontSize: 18,
              fontWeight: FontWeight.bold,
            ),
          ),
        ),
      ),
    Failure(:final error) => Card(
        color: Colors.red.shade50,
        child: Padding(
          padding: const EdgeInsets.all(16.0),
          child: Text(
            'Error: $error',
            style: TextStyle(
              color: Colors.red.shade900,
              fontSize: 16,
            ),
          ),
        ),
      ),
  },

```