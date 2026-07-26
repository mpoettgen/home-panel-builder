# AGENTS.md

This repository is a .NET-based application platform focused on maintainability, readability, and clear architecture.

## Project context
- Use .NET for application and library code.
- Prefer Blazor for web UI development.
- Use NUnit for automated unit tests.
- Follow the Constraint Model in assertions, for example:
  - `Assert.That(actual, Is.EqualTo(expected));`
  - `Assert.That(actual, Is.Not.Null);`
  - `Assert.That(collection, Has.Count.EqualTo(3));`

## General coding guidelines
- Write code that is easy to read and easy to maintain.
- Prefer explicit, descriptive names over short abbreviations.
- Keep methods small and focused on a single responsibility.
- Favor clarity over cleverness.
- Follow consistent formatting and indentation.
- Avoid unnecessary abstractions.

## C# style
- Do not use `var`.
- Follow Microsoft naming conventions.
- Use explicit types for local variables, method return values, fields, and properties.
- Prefer `readonly` for fields that are assigned once.
- Use `private readonly` for dependencies injected into classes.
- Use `sealed` where appropriate for small, closed implementations.
- Prefer `record` for immutable data containers when it improves clarity.
- Prefer asynchronous implementations where meaningful.
- Do not add an `Async` suffix to asynchronous methods.
- Use `async` and `await` consistently for asynchronous operations.
- Prefer `string` interpolation or `string.Format` over concatenation when readability benefits from it.

## Blazor conventions
- Keep components focused and reusable.
- Prefer small, presentational components where practical.
- Use dependency injection for services and shared state.
- Keep UI logic and business logic separated where possible.
- Use clear component names and explicit parameter types.
- Avoid putting too much logic directly into Razor markup.

## Testing guidelines
- Write unit tests with NUnit.
- Use the Constraint Model for assertions.
- Test behavior and outcomes rather than implementation details.
- Keep test names descriptive and intention-revealing.
- Prefer small, focused tests over large, multi-purpose tests.
- Use explicit test data and avoid hidden state where possible.

## Example test style
```csharp
[Test]
public void WhenAddingTwoNumbers_ThenTheResultIsCorrect()
{
    int left = 2;
    int right = 3;

    int result = left + right;

    Assert.That(result, Is.EqualTo(5));
}
```

## Repository expectations
- Preserve existing structure unless there is a strong reason to change it.
- Prefer incremental, understandable changes over large rewrites.
- Document non-obvious decisions when they affect maintainability.
- When adding new features, keep the implementation aligned with the repository’s emphasis on readability and maintainability.

## Working habits for agents
- Prefer minimal, targeted edits.
- Do not introduce new dependencies unless they are clearly justified.
- Avoid style-only changes that would make the diff noisy.
- When uncertain, favor simple and explicit solutions over clever ones.
