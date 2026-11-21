---
name: ahr-assurance
description: Guides AI agents and developers in writing, converting and reviewing .NET tests as Ahr.Assurance scenarios with xUnit v2 or v3, NUnit 3.14 to 5, MSTest 3 or 4, or TUnit - inline and reusable steps, assertions and exception checks, parameterised tests, business context, evidence and reports.
license: Apache-2.0
---

# Ahr.Assurance Agent Skill

This skill guides writing, converting and reviewing .NET tests as Ahr.Assurance scenarios. A scenario names the guarantee being checked, wraps the code the test already has in named Given, When and Then steps, and leaves evidence that `dotnet assurance report` turns into an offline report. Tests keep their framework, runner and assertions.

The skill identity is `ahr-assurance`. It documents the Ahr.Assurance packages and doesn't carry a version of its own.

Further reading. For anything this skill doesn't cover, read the matching page:

- Full docs: https://ahrconsulting.github.io/assurance/
- Getting started: https://ahrconsulting.github.io/assurance/docs/getting-started/
- Scenarios: https://ahrconsulting.github.io/assurance/docs/scenarios/
- The CLI: https://ahrconsulting.github.io/assurance/docs/cli/
- Adapters: https://ahrconsulting.github.io/assurance/docs/adapters/
- Evidence: https://ahrconsulting.github.io/assurance/docs/evidence/
- Running in CI: https://ahrconsulting.github.io/assurance/docs/ci/
- API reference: https://ahrconsulting.github.io/assurance/api/Ahr.Assurance.Execution/

---

## Set up a test project

Install the CLI as a local .NET tool:

```bash
dotnet tool install Ahr.Assurance.Cli --version 0.1.0
```

Then, in the test project's folder or once in the solution's folder:

```bash
dotnet assurance init --framework xunit-v3
```

- `init` writes `ahr-assurance.json` and adds the adapter at the CLI's version.
- The other names are `xunit-v2`, `nunit`, `mstest` (MSTest 4), `mstest-v3` and `tunit`. Add `playwright` beside any of them, such as `--framework xunit-v3,playwright`.
- `init` with no `--framework` reads the project's framework and prints the command for it, without adding a package.
- `dotnet test` records the evidence, and `dotnet assurance report` builds the report from it.

To add an adapter by hand instead, reference its package, then run `dotnet assurance init` without `--framework`:

```bash
dotnet add package Ahr.Assurance.Xunit.v3 --version 0.1.0
dotnet assurance init
```

## 1. Packages

| Framework | Package | What differs |
|---|---|---|
| xUnit v3 | `Ahr.Assurance.Xunit.v3` | Nothing |
| xUnit v2 | `Ahr.Assurance.Xunit.v2` | End with `.VerifyAsync(output)`, the test's `ITestOutputHelper` |
| NUnit 3.14, 4 or 5 | `Ahr.Assurance.NUnit` | Nothing |
| MSTest 4 | `Ahr.Assurance.MSTest` | On 4.0 to 4.4, end with `.VerifyAsync(TestContext)` |
| MSTest 3 | `Ahr.Assurance.MSTest.v3` | End with `.VerifyAsync(TestContext)` |
| TUnit | `Ahr.Assurance.TUnit` | Checks that assert take an `async` lambda |

- Each adapter brings `Ahr.Assurance` and its analyzers. With `<ImplicitUsings>enable</ImplicitUsings>`, write no Assurance `using` lines.
- `Ahr.Assurance.Playwright` adds screenshots and traces under any adapter.
- `Ahr.Assurance.Cli` is `dotnet assurance init` and `dotnet assurance report`. See [Set up a test project](#set-up-a-test-project).

## 2. When to write a scenario

- **Scenario:** behaviour the team has committed to, which someone outside the team would recognise. That covers business rules, journeys and a library's guarantees.
- **Plain test:** technical plumbing, small helpers, and behaviour still being shaped. Plain tests run beside scenarios.
- **Inline first.** Move steps into a steps class only when they appear in two or three tests.

## 3. Inline scenarios

<!-- snippet: first-scenario -->
```csharp
[Fact]
public Task ASavedDocumentOpensOfflineAsAScenario()
{
    Document expected = new("handbook", "Offline-ready field handbook");
    FakeDocumentSource source = new(expected);
    FakeInMemoryDocumentCache cache = new();
    DocumentReader reader = new(new FakeOfflineConnectivity(), source, cache);
    DocumentReadResult? result = null;

    return Scenario.Define("A returning offline reader opens a saved handbook")
        .Given("An offline reader with a saved document", () => cache.Seed(expected))
        .When("The reader opens the handbook", async token => result = await reader.OpenAsync("handbook", token))
        .Then("The saved document is returned", () => Assert.Equal(expected, result?.Document))
        .And("The web source is not contacted", () => Assert.Equal(0, source.DownloadCalls))
        .VerifyAsync();
}
```

- The scenario name states the guarantee. Step names are plain sentences saying what the step means, not "Arrange" or "Act".
- Steps go `Given`, `When`, then `Then`. `Given` is optional, a `When` is required, and the last step is a `Then` or a throws step. `And` continues the step before it.
- A workflow repeats When and Then. Use it only when each action starts from the state the one before it left.
- Always return or await `VerifyAsync()`. Nothing runs without it.
- Declare locals before `Define`; steps assign and read them. When converting a test, keep its result local and assign it in the When, as above; read it with `?.` in the checks.
- A When that returns a value hands it to the checks that take its type: `.When("The reader opens the handbook", token => reader.OpenAsync("handbook", token)).Then("The expected document is returned", (DocumentReadResult result) => ...)`. Prefer it in a new scenario whose value one or two checks read. What a Given returns isn't kept.
- A call that takes only the token is passed as a method group: `.When("...", client.GetOrdersAsync)`.

## 4. Assertions and exceptions

- Use the framework's ordinary assertions in any step. xUnit, NUnit, MSTest and TUnit asserts, Shouldly, FluentAssertions and Playwright's `Expect(...)` are recognised under every adapter.
- A failed assertion in Then fails that check, and the later checks still run. In Given or When it stops the scenario.
- `Assert.Skip`, `Assert.Ignore` or `Assert.Inconclusive` inside a step skips the test. Any other exception is an execution error.
- Attach long output in the step that produces it: `Attach.Text("output.txt", output, AttachmentKeep.OnFailure)`.

Write the step that should throw with `WhenThrows<T>`, and check the exception with a lambda that takes it:

<!-- snippet: exception -->
```csharp
return Scenario.Define("A charge of nothing is refused")
    .Given("A basket", () => basket = new Basket())
    .WhenThrows<ArgumentOutOfRangeException>("Nothing is charged", () => basket.Charge(0m))
    .Then("The refusal names the amount", (ArgumentOutOfRangeException refusal) => Assert.Equal("amount", refusal.ParamName))
    .VerifyAsync();
```

- It passes only for exactly `T`. For `T` or a derived type, use `WhenThrowsAny<T>`.
- Several expected exceptions: `.WhenThrows<A>(...).AndThrows<B>(...)`, and checks that take `A` and `B`.

## 5. Parameterised tests

Use the framework's data attributes unchanged. Keep the scenario name the same for every row, with no row data in it. Each row records its arguments as an example and gets its own ID, and the report shows the rows as one journey with an examples table.

## 6. Reusable steps

<!-- snippet: reusable-scenario -->
```csharp
[Fact]
public Task AReaderOpensASavedDocument() =>
    Scenario.Define<ReadingContext>("A reader opens a saved document")
        .Given(ReadingSteps.ASavedDocument("handbook"))
        .When(ReadingSteps.TheReaderOpens("handbook"))
        .Then(ReadingSteps.TheDocumentIsShown("handbook"))
        .VerifyAsync();
```

<!-- snippet: steps-class -->
```csharp
internal sealed class ReadingSteps : ScenarioSteps<ReadingContext>
{
    public static ScenarioAction<ReadingContext> ASavedDocument(string id) =>
        CreateActionStep($"'{id}' is saved on the device", context => context.Cache.Seed(new Document(id, "Field handbook")));

    public static ScenarioAction<ReadingContext> TheReaderOpens(string id) =>
        CreateActionStep($"The reader opens '{id}'", async (context, token) =>
            context.Result = await new DocumentReader(context.Connectivity, context.Source, context.Cache).OpenAsync(id, token));

    public static ScenarioExpectation<ReadingContext> TheDocumentIsShown(string id) =>
        CreateAssertionStep($"'{id}' is shown", context => Assert.Equal(id, context.Result?.Document?.Id));
}
```

- `ScenarioSteps<TContext>` is the one base class under every framework. Step properties are `static`, and a step that takes values is a `static` method that builds its own name.
- A context is a plain class holding the state and fakes a scenario starts from. Each run gets a fresh one, and it's disposed after the last step.
- Never write an empty Given. When the context already sets up the start, begin with a Given that names it.
- `.WithValue(code)` records an input that the step name leaves out, for the report's developer view.
- Steps that cut across features go in a steps class written for an interface. Every scenario whose context implements it can use them alongside its own steps; one that doesn't fails to compile:

<!-- snippet: shared-steps -->
```csharp
public interface IDevice
{
    IConnectivity Connectivity { get; set; }
}

internal sealed class DeviceSteps : ScenarioSteps<IDevice>
{
    public static ScenarioAction<IDevice> TheDeviceIsOffline { get; } =
        CreateActionStep("The device is offline", device => device.Connectivity = new FakeOfflineConnectivity());

    public static ScenarioAction<IDevice> TheDeviceIsOnline { get; } =
        CreateActionStep("The device is online", device => device.Connectivity = new FakeOnlineConnectivity());
}
```

## 7. Business context

<!-- snippet: business-context -->
```csharp
private static readonly BusinessContext _offlineDocuments = BusinessContext
    .Define("offline-document-access", "Offline document access", "Keep documents available when connectivity changes")
    .Capability("read-saved-documents", "Read a saved document without a network")
    .Outcome("saved-content-available", "Saved content is returned to the reader")
    .Rule("offline-does-not-fetch", "Offline reads never contact the web source")
    .Risk("offline-content-missing", "A returning reader cannot access a saved document")
    .Reference("jira", "DOC-42", new Uri("https://example.atlassian.net/browse/DOC-42"))
    .Build();
```

- Pass the context to `Define`. One definition per ID in each test project.
- Checks that only developers read, such as argument guards, go in a context of their own, such as `technical`. Leave that context out of stakeholder reports with `report.excludeContexts` in `ahr-assurance.json`, or `--exclude-context`.

## 8. Browser tests

<!-- snippet: playwright -->
```csharp
public static async Task AnEmptyBasketCostsNothing(IPage page)
{
    using IDisposable capture = AssurancePlaywright.Capture(page);
    _ = await Scenario.Define("An empty basket costs nothing")
        .Given("An empty basket", () => page.GotoAsync("/basket"))
        .When("The total is worked out", () => page.ClickAsync("#update-total"))
        .Then("It is zero", async () => Assert.Equal("£0.00", await page.InnerTextAsync("#total")))
        .VerifyAsync();
}
```

Create the capture before `Scenario.Define`, and keep it until `VerifyAsync()` has finished.

## 9. Reports

```bash
dotnet test
```

```bash
dotnet assurance report
```

`dotnet test` records the evidence, and `dotnet assurance report` builds the report: it reports the newest run in a test project's folder, or each project's newest run in a solution's folder. `--developer` adds failure details. The exit code is 2 when the report shows failures and 1 when the command couldn't run.

In CI, run the report step even when `dotnet test` fails, and set `ASSURANCE_RUN_ID` to one value across shards so they write to the same run. Assurance sits beside TRX or other test results and doesn't change them.

## 10. Mistakes the analyzers report

| Rule | Mistake | Instead |
|---|---|---|
| AHRA101 | `() => pending = StartAsync()` makes the step wait for the task | `() => { pending = StartAsync(); }` |
| AHRA102 | A chain without `VerifyAsync()`, or `VerifyAsync()` not awaited or returned | Return or await `VerifyAsync()` |
| AHRA103 | `using var x = ...; return ...VerifyAsync();` disposes `x` before the steps run | Make the test `async` and `await` |
| AHRA105 | Steps out of order, or no When | Given (optional), When, Then |
| AHRA106 | Two scenarios in a project with the same name and no ID | Distinct names, or `id:` |
| AHRA107 | An inline scenario stored in a field and reused | Define it inside each test |
| AHRA108 | Instance step properties | `static` step properties |

Also: never verify the same definition twice, don't assert outside the scenario's steps, and keep one `VerifyAsync()` per test.
