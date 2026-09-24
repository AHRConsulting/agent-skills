---
name: ahr-foundation
description: Guides AI agents and developers in using Ahr.Foundation for robust Railway-Oriented Programming (ROP) in .NET with Result, Result<T>, Result<T, TError>, Option<T>, and TaskCompositionExtensions.
license: Apache-2.0
---

# Ahr.Foundation Agent Skill

This skill provides expert instructions and architectural guidelines for generating, refactoring, and auditing C# code that leverages `Ahr.Foundation` for explicit functional error handling and option modeling.

The skill identity is `ahr-foundation`; it documents the `Ahr.Foundation` NuGet package and does not carry an independent semantic version. Install the stable package with:

```bash
dotnet add package Ahr.Foundation --version 0.1.0
```

---

## 1. Core Primitives & Philosophy

`Ahr.Foundation` is a platform-neutral library with no runtime package dependencies, built on strict invariants. Its analyzer is delivered as a separate development asset:

1. **Explicit Initialization**:
   - `Result`, `Result<T>`, and `Result<T, TError>` **must not** be created via `default` or parameterless `new()`.
   - Default-constructed `Result`, `Result<T>`, and `Result<T, TError>` values throw `InvalidOperationException` if inspected or composed.
   - `Option<T>.None` is intentionally represented by the safe default value.
   - Always instantiate via factories: `Result.Success()`, `Result.Failure(error)`, `Result<T>.Success(value)`, `Result<T>.Failure(error)`, or extension methods (`val.ToSuccess()`, `"error".ToFailure()`).
   - The Roslyn analyzer rule `AHRF001` detects default construction at compile-time.

2. **Branch Access Rules**:
   - Accessing `.Value` on a failed result throws `InvalidOperationException`, including custom-error results.
   - Accessing `.Error` on a successful result throws `InvalidOperationException`, including custom-error results.
   - Accessing `.Value` on `Option<T>.None` throws `InvalidOperationException`.
   - Prefer pattern matching (`Match`, `MatchAsync`) or monadic combinators (`Bind`, `Map`, `MapError`, `Tap`, `Ensure`, `Where`) over direct property access.

3. **Platform & Framework Agnostic**:
   - `Ahr.Foundation` has no runtime package dependencies and no UI or platform-specific coupling.
   - Can be used in Console apps, Microservices, Web APIs, Desktop, Mobile, and Cloud Functions.

---

## 2. API Quick Reference & Combinator Recipes

### Creating Results and Options

```csharp
using Ahr.Foundation;

// Payload-free Result
Result ok = Result.Success();
Result fail = Result.Failure(new Error("Database connection lost"));
Result failFromStr = "Invalid input".ToFailure();

// Generic Result<T>
Result<int> numSuccess = 42.ToSuccess();
Result<int> numFail = "Must be positive".ToFailure<int>();

// Custom Error Result<T, TError>
Result<User, DomainError> customRes = Result<User, DomainError>.Success(user);
Result<User, DomainError> customFail = Result<User, DomainError>.Failure(DomainError.NotFound);

// Option<T>
Option<string> some = Option<string>.Some("value");
Option<string> none = Option<string>.None; // or default
Option<string> safe = nullableValue.ToOption();
Option<string> firstMatch = list.FirstOrNone(x => x.StartsWith("A"));
Option<int> filtered = Option<int>.Some(21).Where(x => x > 0);
Result<int> converted = filtered.ToResult(new Error("Value is missing"));

// Extracting values without branching on IsSuccess/IsSome
int value = numSuccess.GetValueOrDefault(); // 42
int missing = numFail.GetValueOrDefault(); // 0
int withFallback = numFail.GetValueOrDefault(-1); // -1

// Falling back to an alternative Result/Option (not just a plain value)
Result<int> localOrDefault = numFail.OrElse(Result<int>.Success(0));
Result<int> localOrRemote = numFail.OrElse(() => LookupRemote());
Result<int> localOrRemoteAsync = await numFail.OrElseAsync(() => LookupRemoteAsync());
Option<string> someOrElse = none.OrElse(some);
```

`TaskCompositionExtensions` also supports `Task<Option<T>>` and `Task<Result<T, TError>>`
receivers, including their corresponding async mapping, binding, error mapping, filtering,
validation, and tap operations. It also provides `OrElseAsync(Func<Task<fallback>>)` for
`Task<Result>`, `Task<Result<T>>`, `Task<Result<T, TError>>`, and `Task<Option<T>>`, letting
`OrElseAsync` be chained directly off an unawaited task, e.g.
`await repository.FindLocalAsync(id).OrElseAsync(() => repository.FindRemoteAsync(id))`.

### Pipeline Composition (Railway-Oriented Programming)

```csharp
// Synchronous pipeline
public Result<Order> ProcessOrder(OrderRequest request)
{
    return ValidateRequest(request)
        .Bind(req => CheckInventory(req))
        .Ensure(order => order.TotalAmount > 0, new Error("Order total must be positive"))
        .Tap(order => LogOrderCreated(order.Id))
        .TapError(err => LogError(err.Message));
}

// Asynchronous pipeline with TaskCompositionExtensions
public async Task<Result<OrderConfirmation>> ProcessOrderPipelineAsync(OrderRequest request)
{
    return await ValidateRequestAsync(request)
        .BindAsync(req => CheckInventoryAsync(req))
        .BindAsync(order => ProcessPaymentAsync(order))
        .MapAsync(payment => new OrderConfirmation(payment.TransactionId, payment.Amount))
        .TapAsync(conf => NotifyCustomerAsync(conf))
        .TapErrorAsync(err => LogAlertAsync(err));
}
```

### LINQ Query Syntax

`Select` and `SelectMany` are declared as aliases over `Map` and `Bind`, so query comprehensions
work and short-circuit on the first failure exactly as the equivalent method chain does.

```csharp
// Each clause sees the values bound before it; the query stops at the first failure.
Result<decimal> total =
    from user in FindUser(userId)
    from cart in LoadCart(user)
    from priced in PriceCart(cart)
    select priced.Total;

// Equivalent method syntax
Result<decimal> same = FindUser(userId)
    .Bind(user => LoadCart(user)
        .Bind(cart => PriceCart(cart)
            .Map(priced => priced.Total)));
```

Guidance for agents:

- Prefer **method syntax** (`Map`/`Bind`) by default; it is the primary API and reads well for
  short chains.
- Reach for **query syntax** when three or more dependent steps each need earlier values still in
  scope, which otherwise forces deep `Bind` nesting.
- `Select`/`SelectMany` are available on `Result<T>`, `Result<T, TError>`, and `Option<T>`.
- There is **no** async query syntax. For `Task<Result<T>>` pipelines use `BindAsync`/`MapAsync`.
- Do not mix the two styles within a single expression.

### Pattern Matching

```csharp
// Value-returning Match
string message = result.Match(
    onSuccess: user => $"Welcome, {user.Name}!",
    onFailure: error => $"Error: {error.Message}");

// Void / Action Match
result.Match(
    onSuccess: user => SendWelcomeEmail(user),
    onFailure: error => ReportToTelemetry(error));
```

### Boundary Exception Handling: Result.Try / Result.TryAsync

`Result.Try`/`Result.TryAsync` convert an exception thrown by an operation into a `Result` failure,
replacing a repetitive try/catch block at I/O and interop boundaries.

```csharp
// Payload-free, built-in Error
Result saved = Result.Try(() => repository.Save(entity));

// Result<T>, built-in Error, synchronous and asynchronous
Result<Order> order = Result.Try(() => repository.GetOrder(orderId));
Result<Customer> customer = await Result.TryAsync(() => repository.GetCustomerAsync());

// Result<T, TError>, custom error factory required
Result<Customer, DomainError> customCustomer = await Result.TryAsync(
    () => repository.GetCustomerAsync(),
    ex => DomainError.FromException(ex));
```

Rules that apply to every overload:

- `OperationCanceledException` (including `TaskCanceledException`) always propagates unchanged; it
  is never converted into a `Failure`.
- Built-in-`Error` overloads (`Try`, `Try<T>`, `TryAsync`, `TryAsync<T>`) call
  `Error.FromException` automatically. Custom-error overloads (`Try<T, TError>`,
  `TryAsync<T, TError>`) require an explicit `Func<Exception, TError>` error factory.
- A `null` operation or error factory, or a `null` value/error/task produced by the operation, is
  not swallowed — it propagates as a real exception instead of silently becoming a `Failure`.
- The `AHRF002` analyzer (Info) flags a manual try/catch that only constructs a Result success in
  the try block and a Result failure in the catch block, and suggests `Result.Try`/`Result.TryAsync`
  instead.
- The `AHRF003` analyzer (Warning) flags a synchronous `Result.Try` call whose delegate discards an
  awaitable (`Task`/`Task<T>`/`ValueTask`/`ValueTask<T>`), e.g. `Result.Try(() =>
  repository.GetCustomerAsync())`. This silently runs the operation fire-and-forget: exceptions
  thrown after the first `await` are never caught by `Try`. Use the matching `Result.TryAsync`
  overload instead.
- The `AHRF004` analyzer (Warning) flags an async delegate explicitly typed or cast as `Action` and
  passed to `Result.Try`, e.g. `Result.Try((Action)(async () => await op()))`. Such a delegate is
  async-void: `Try`'s `catch` block only observes exceptions thrown synchronously before the first
  `await`, so anything thrown afterward escapes unobserved. Note an inline async lambda passed
  directly to `Result.Try` (no explicit `Action` cast) instead resolves via overload resolution to
  `Try<T>` and is flagged by `AHRF003`; `AHRF004` only fires for the explicitly-`Action`-typed case.
  It does not perform data-flow analysis to trace a delegate variable back to the async lambda
  assigned to it.

---

## 3. Best Practices for AI Agents

1. **Never suppress AHRF001**: Always replace `default(Result)`, `default(Result<T>)`, `default(Result<T, TError>)`, or parameterless result construction with explicit factories. `default(Option<T>)` is valid and represents `None`.
2. **Propagate Errors Early**: Use `.Bind()` and `.BindAsync()` to stop execution on first failure without throwing exceptions.
3. **Keep Exceptions at the Boundary**: Convert infrastructure exceptions to domain `Error` instances via `Error.FromException(ex)` or `ex.ToFailure<T>()`. Null payloads, errors, delegates, and task results are rejected by the relevant factory or composition method rather than silently converted.
4. **Use Options for Absence, Results for Failure**: If "not found" is normal/expected, return `Option<T>`. If absence signifies an error condition that stops an operation, return `Result<T>`.
5. **Choose the Clearest Composition Style**: Use `Map`/`Bind` for short chains, and LINQ query syntax when several dependent steps need earlier values in scope. Never use query syntax for async pipelines.
6. **Prefer Result.Try/TryAsync over manual try/catch**: When converting an operation that can throw
   into a `Result`, use `Result.Try`/`Result.TryAsync` instead of a manual try/catch that constructs
   `Success`/`Failure`. Heed `AHRF002` when it fires.
7. **Never pass an awaitable-returning delegate to a synchronous `Result.Try` overload**: If the
   operation returns `Task`/`Task<T>`/`ValueTask`/`ValueTask<T>`, use `Result.TryAsync` so exceptions
   thrown during the awaited operation are actually captured. Heed `AHRF003` when it fires.
8. **Never explicitly type/cast an async delegate as `Action` when passing it to `Result.Try`**: This
   creates an async-void delegate whose post-`await` exceptions are never captured. Heed `AHRF004`
   when it fires.
9. **Prefer `GetValueOrDefault()` over `Match`/`IsSuccess` when a default or fallback value is
   acceptable**: Use `GetValueOrDefault()`/`GetValueOrDefault(fallback)` to read a success payload
   without branching, reserving `Match`/`Bind`/`Ensure` for cases that need to observe or react to
   the failure itself.
10. **Prefer `OrElse`/`OrElseAsync` over `Match` when falling back to another `Result`/`Option`**:
    Use `OrElse(fallback)`/`OrElse(Func<fallback>)`/`OrElseAsync(Func<Task<fallback>>)` when the
    fallback is itself a `Result`/`Option` (for example, a secondary data source), rather than
    `GetValueOrDefault`, which only extracts a plain payload value. Prefer the lazy/async overload
    over the eager one whenever constructing the fallback has any cost, since it is only invoked on
    failure/`None`.
