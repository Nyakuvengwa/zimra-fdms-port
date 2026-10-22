# Plan: C# NuGet port of zimra-fdms

## 0. Document rules

This document uses ASD-STE100 Simplified Technical English.

This document contains zero em dashes.

Every file in the `dotnet` folder uses CRLF line endings.

Every file in the `dotnet` folder ends with one CRLF.

Use UTF-8 without a byte order mark.

Use `.gitattributes` to enforce CRLF.

Use `.editorconfig` to enforce CRLF and one final newline.

## 1. Decisions

- Port target: a C# NuGet package.
- Package ID: `Zimra.Fdms`.
- Target frameworks: `net8.0` and `net10.0`.
- Location: the `dotnet` folder in this repository.
- Scope: full feature parity with the TypeScript SDK.
- Scope: core, HTTP, mutual TLS, storage, journal, offline queue, registration, CLI, MCP server, simulator and conformance tools.
- Design: fresh and idiomatic C#.
- Design: separate injected services.
- Design: no `FiscalDevice` class.
- Design: no compatibility facade.
- Contract: FDMS wire JSON and canonical signing strings must match exactly.
- Contract: the published vectors are the acceptance contract.
- Documentation: every method has an XML summary.
- Documentation: every public method has full parameter, return and exception documentation.
- Documentation: all prose uses ASD-STE100.
- Documentation: zero em dashes in all files.

Out of scope:

- The React Native bridge.
- The Android and MAUI keystore adapter.
- Legacy TypeScript 0.3.x queue migration.
- FDMS endpoints that the TypeScript SDK does not use.

The TypeScript SDK stays unchanged. The C# port is additive.

## 2. Solution layout

```text
dotnet/
  PLAN.md
  .gitattributes
  .editorconfig
  Zimra.Fdms.sln
  global.json
  Directory.Build.props
  Directory.Packages.props
  README.md
  DOCUMENTATION.md
  src/
    Zimra.Fdms/
    Zimra.Fdms.Mcp/
    Zimra.Fdms.Cli/
    Zimra.Fdms.Simulator/
    Zimra.Fdms.Simulator.Cli/
    Zimra.Fdms.Conformance/
    Zimra.Fdms.Testing/
  tests/
    Zimra.Fdms.Tests/
    Zimra.Fdms.Simulator.Tests/
    Zimra.Fdms.Cli.Tests/
    Zimra.Fdms.Mcp.Tests/
    Zimra.Fdms.Conformance.Tests/
  samples/
    Zimra.Fdms.Sample/
    Zimra.Fdms.HardwareSignerSample/
```

Use central package management.

Use `Directory.Build.props` for shared build settings.

## 3. Package map

| Package | Type | Command | Contents |
| --- | --- | --- | --- |
| `Zimra.Fdms` | Library | None | Core engine, models, signing, crypto, HTTP, storage, profile, queue |
| `Zimra.Fdms.Mcp` | Library | None | MCP server factory and tools |
| `Zimra.Fdms.Cli` | Tool | `zimra-fdms` | CLI and `mcp` subcommand |
| `Zimra.Fdms.Simulator` | Library | None | Local FDMS simulator and CA |
| `Zimra.Fdms.Simulator.Cli` | Tool | `zimra-fdms-simulator` | Simulator entry point |
| `Zimra.Fdms.Conformance` | Tool | `zimra-fdms-conformance` | Vector runner and responder |
| `Zimra.Fdms.Testing` | Library | None | Fakes, builders and test helpers |

Keep `Zimra.Fdms` free of third-party runtime dependencies.

Use the .NET base class library for crypto, HTTP and JSON.

Use `Microsoft.Extensions.DependencyInjection.Abstractions` for service registration.

## 4. Architecture

### 4.1 Remove FiscalDevice

Do not create a `FiscalDevice` class.

Do not create a compatibility facade.

Replace the class with focused services.

Each service has one job.

Each service has a small interface.

Each service has a public constructor.

Each service depends on interfaces.

### 4.2 Services

| Interface | Responsibility | Main methods |
| --- | --- | --- |
| `IDeviceInfo` | Device endpoints | `GetConfigAsync`, `GetStatusAsync`, `PingAsync` |
| `IFiscalDay` | Fiscal day lifecycle | `OpenAsync`, `CloseAsync` |
| `IReceipts` | Receipt lifecycle | `PrepareAsync`, `SubmitAsync`, `SubmitPreparedAsync`, `GetQrData` |
| `ICertificates` | Certificate renewal | `RenewAsync` |
| `IDeviceState` | State and pending marker | `Snapshot`, `Restore`, `GetLastIssuedGlobalNoAsync`, `GetPendingReceiptAsync`, `DiscardPendingAsync`, `ClearAsync` |
| `IReconciler` | Startup recovery | `ReconcileAsync` |
| `IDeviceRegistrar` | Device registration | `RegisterAsync` |
| `IOfflineReceiptQueue` | Offline sales | `EnqueueAsync`, `SubmitOrEnqueueAsync`, `FlushAsync` |

Use these public interfaces.

Keep these classes internal:

- `FdmsClient`
- `DeviceStateStore`
- `DeviceConfigCache`
- `DeviceOperationGate`

Test internal classes with `InternalsVisibleTo`.

### 4.3 Shared internals

- `FdmsClient`: HTTP paths, headers, JSON and error mapping.
- `DeviceStateStore`: implements `IDeviceState`.
- `DeviceConfigCache`: stores the last `GetConfig` response.
- `DeviceOperationGate`: one shared `SemaphoreSlim`.
- `ServerCorrectedClock`: learns the FDMS clock offset.

All services use the same `DeviceStateStore` and the same gate.

`IFiscalDay`, `IReceipts` and `IReconciler` acquire the gate.

`IDeviceInfo` does not acquire the gate.

### 4.4 Service graph

- `DeviceInfo` depends on `IFdmsApi` and `IDeviceConfigCache`.
- `FiscalDay` depends on `IFdmsApi`, `IDeviceState`, `ISigner`, `IClock` and `IDeviceOperationGate`.
- `Receipts` depends on `IFdmsApi`, `IDeviceState`, `ISigner`, `IClock`, `IDeviceConfigCache` and `IDeviceOperationGate`.
- `Certificates` depends on `IFdmsApi`, `ISigner` and `ICsBuilder`.
- `Reconciler` depends on `IFdmsApi`, `IDeviceState` and `IDeviceOperationGate`.
- `OfflineReceiptQueue` depends on `IReceipts`, `IDeviceState`, `IJournal` and `IClock`.
- `DeviceRegistrar` depends on `IFdmsApi`, `ISigner` and `ICsBuilder`.

The graph is acyclic.

### 4.5 Dependency injection

Use standard .NET dependency injection.

Add one registration extension.

```csharp
public static class ZimraFdmsServiceCollectionExtensions
{
    public static IServiceCollection AddZimraFdms(
        this IServiceCollection services,
        DeviceIdentity identity,
        FdmsDeviceOptions options,
        FdmsDeviceDependencies dependencies);
}
```

Register these as singletons:

- `DeviceIdentity`
- `FdmsDeviceOptions`
- `ISigner`
- `IFdmsTransport`
- `IStorage`
- `IJournal`
- `IClock`
- `ServerCorrectedClock`
- `IDeviceState`
- `IDeviceInfo`
- `IFiscalDay`
- `IReceipts`
- `ICertificates`
- `IReconciler`
- `IDeviceRegistrar`
- `IOfflineReceiptQueue`
- `IFdmsApi`

Use one service provider per device profile.

Document this rule.

The container disposes the transport if it implements `IAsyncDisposable`.

### 4.6 Usage example

```csharp
var services = new ServiceCollection();

services.AddZimraFdms(
    identity,
    new FdmsDeviceOptions
    {
        Environment = FdmsEnvironment.Test,
        Storage = new FileStorage("./.zimra"),
        Journal = new FileJournal("./.zimra")
    },
    new FdmsDeviceDependencies
    {
        Signer = new PemSigner(privateKeyPem),
        Transport = new MtlsHttpTransport(
            new MtlsIdentity(certificatePem, privateKeyPem))
    });

await using var provider = services.BuildServiceProvider();

var info = provider.GetRequiredService<IDeviceInfo>();
var day = provider.GetRequiredService<IFiscalDay>();
var receipts = provider.GetRequiredService<IReceipts>();
var reconciler = provider.GetRequiredService<IReconciler>();

await reconciler.ReconcileAsync(cancellationToken);
await info.GetConfigAsync(cancellationToken);
await day.OpenAsync(cancellationToken: cancellationToken);

var sale = await receipts.SubmitAsync(input, cancellationToken);

await day.CloseAsync(cancellationToken: cancellationToken);
```

The caller resolves only the services it needs.

## 5. Public contracts

### 5.1 Service interfaces

```csharp
public interface IDeviceInfo
{
    Task<GetConfigResponse> GetConfigAsync(
        CancellationToken cancellationToken = default);

    Task<GetStatusResponse> GetStatusAsync(
        CancellationToken cancellationToken = default);

    Task<PingResponse> PingAsync(
        CancellationToken cancellationToken = default);
}

public interface IFiscalDay
{
    Task<OpenDayResponse> OpenAsync(
        OpenDayOptions? options = null,
        CancellationToken cancellationToken = default);

    Task<CloseDayResponse> CloseAsync(
        CloseDayOptions? options = null,
        CancellationToken cancellationToken = default);
}

public interface IReceipts
{
    Task<PreparedReceipt> PrepareAsync(
        ReceiptInput input,
        CancellationToken cancellationToken = default);

    Task<SubmittedReceipt> SubmitAsync(
        ReceiptInput input,
        CancellationToken cancellationToken = default);

    Task<SubmittedReceipt> SubmitPreparedAsync(
        PreparedReceipt prepared,
        CancellationToken cancellationToken = default);

    string? GetQrData(PreparedReceipt prepared);
}

public interface ICertificates
{
    Task<CertificateRenewal> RenewAsync(
        ISigner? signer = null,
        CancellationToken cancellationToken = default);
}

public interface IDeviceState
{
    FiscalDayState? Snapshot { get; }

    void Restore(FiscalDayState state);

    Task<long?> GetLastIssuedGlobalNoAsync(
        CancellationToken cancellationToken = default);

    Task<PreparedReceipt?> GetPendingReceiptAsync(
        CancellationToken cancellationToken = default);

    Task DiscardPendingAsync(
        CancellationToken cancellationToken = default);

    Task ClearAsync(
        CancellationToken cancellationToken = default);
}

public interface IReconciler
{
    Task<ReconcileResult> ReconcileAsync(
        CancellationToken cancellationToken = default);
}

public interface IDeviceRegistrar
{
    Task<RegisteredDevice> RegisterAsync(
        DeviceIdentity identity,
        string activationKey,
        ISigner signer,
        RegistrationOptions? options = null,
        CancellationToken cancellationToken = default);
}

public interface IOfflineReceiptQueue
{
    int Count { get; }

    TimeSpan? OldestPendingAge { get; }

    Task<PreparedReceipt> EnqueueAsync(
        ReceiptInput input,
        CancellationToken cancellationToken = default);

    Task<SubmittedReceipt?> SubmitOrEnqueueAsync(
        ReceiptInput input,
        CancellationToken cancellationToken = default);

    Task<FlushResult> FlushAsync(
        CancellationToken cancellationToken = default);
}
```

### 5.2 Dependency interfaces

```csharp
public interface ISigner
{
    Task<byte[]> SignAsync(
        ReadOnlyMemory<byte> data,
        CancellationToken cancellationToken = default);

    Task<byte[]> GetPublicKeySpkiAsync(
        CancellationToken cancellationToken = default);
}

public interface IFdmsTransport
{
    Task<TransportResponse> SendAsync(
        TransportRequest request,
        CancellationToken cancellationToken = default);
}

public interface IStorage
{
    Task<string?> GetAsync(
        string key,
        CancellationToken cancellationToken = default);

    Task SetAsync(
        string key,
        string value,
        CancellationToken cancellationToken = default);

    Task DeleteAsync(
        string key,
        CancellationToken cancellationToken = default);
}

public interface IJournal
{
    Task AppendAsync(
        string record,
        CancellationToken cancellationToken = default);

    Task<IReadOnlyList<JournalEntry>> ReadFromAsync(
        long cursor,
        CancellationToken cancellationToken = default);

    Task CommitAsync(
        long cursor,
        CancellationToken cancellationToken = default);

    Task<long> GetCommittedAsync(
        CancellationToken cancellationToken = default);

    Task ReplaceAsync(
        long cursor,
        string record,
        CancellationToken cancellationToken = default);
}

public interface IDeviceConfigCache
{
    GetConfigResponse? Current { get; }

    void Set(GetConfigResponse config);
}

public interface IDeviceOperationGate
{
    Task<IAsyncDisposable> AcquireAsync(
        CancellationToken cancellationToken = default);
}

public interface IRetryDelay
{
    Task DelayAsync(
        int attempt,
        CancellationToken cancellationToken = default);
}

public interface IRandom
{
    double NextDouble();
}

public interface ICsrBuilder
{
    Task<string> BuildAsync(
        ISigner signer,
        string commonName,
        CancellationToken cancellationToken = default);
}

public interface IClock
{
    DateTimeOffset Now { get; }
}
```

### 5.3 Money and dates

- Use `Money` readonly record struct with `long Cents`.
- Add `Money.FromCents`, `Money.FromMajorUnits` and `Money.Parse`.
- Do not use `double` for money.
- Use `decimal` only for tax percentages.
- Use `FdmsTimestamp` for wire date and time values.
- Store `DateTimeOffset` in `FdmsTimestamp`.
- Format wire dates with `yyyy-MM-ddTHH:mm:ss` in local time.
- Use `DateOnly` for fiscal day dates.
- Use `TimeProvider` inside the clock and retry delay.
- Use `IClock` in services.

### 5.4 Errors

- `FdmsApiException` with `Status`, `Problem`, `OperationId`, `SupportCode` and `Explain()`.
- `DayNotClosableException`.
- `PendingSubmitException`.
- `TransportException`.
- `ProfileException`.
- Port the error catalogue text and support code format.

### 5.5 JSON

- Use `System.Text.Json` with source generation.
- Put `[JsonPropertyName]` on every wire property.
- Use `JsonStringEnumConverter` with exact wire names.
- Add custom converters for `Money`, `FdmsTimestamp` and `FiscalDayCounter`.
- Use invariant culture for all formatting and parsing.
- Add golden tests for request bodies and response parsing.
- Keep null and omitted fields deliberate.
- Match the FDMS contract.

## 6. CancellationToken contract

Use the full name everywhere.

Do not use `ct`.

Use this signature on every async method:

```csharp
CancellationToken cancellationToken = default
```

Apply it to these APIs:

- `IDeviceInfo`
- `IFiscalDay`
- `IReceipts`
- `ICertificates`
- `IDeviceState`
- `IReconciler`
- `IDeviceRegistrar`
- `IOfflineReceiptQueue`
- `ISigner`
- `IFdmsTransport`
- `IStorage`
- `IJournal`
- `ICsrBuilder`
- `IDeviceOperationGate`
- `IRetryDelay`
- `FiscalDayService`
- `ReceiptService`
- `CertificateService`
- `Reconciler`
- `DeviceStateStore`
- `FdmsSimulator.StartAsync`
- `FdmsSimulator.StopAsync`
- `SimulatorCa.IssueFromCsrAsync`
- `SimulatorCa.SignAsync`
- CLI command handlers
- MCP tool handlers
- Conformance runner and responder

Forward the token to every async call:

- `HttpClient.SendAsync`
- `HttpContent.ReadAsStringAsync`
- `JsonSerializer.DeserializeAsync`
- `JsonSerializer.SerializeAsync`
- `Task.Delay`
- `SemaphoreSlim.WaitAsync`
- `FileStream` read and write calls
- `StreamReader` and `StreamWriter`
- `ISigner.SignAsync`
- `IFdmsTransport.SendAsync`
- `IStorage` and `IJournal` calls

Use linked tokens for request timeouts.

Keep the caller token as the parent.

Let `OperationCanceledException` propagate.

Do not wrap it in `TransportException` or `FdmsApiException`.

Call `cancellationToken.ThrowIfCancellationRequested()` before each retry.

Pass the token to the retry delay.

Use `ConfigureAwait(false)` in library code.

Enable `CA2016` and `CA2007` as build errors.

Add a reflection test.

The test checks every public async method in `Zimra.Fdms` and `Zimra.Fdms.Mcp`.

Each method must have a `CancellationToken` parameter named `cancellationToken`.

Add cancellation tests:

- Cancel before the call. Expect `OperationCanceledException`.
- Cancel during a retry delay. Expect a fast stop.
- Cancel during HTTP. Expect `OperationCanceledException`, not `TransportException`.
- Cancel during a file write. Expect the token to reach the file layer.
- Cancel an MCP tool call. Expect the token to reach the service.
- Cancel the CLI with `Ctrl+C`. Expect a clean exit.

## 7. Documentation standard

### 7.1 Documentation scope

Document these items:

1. Every interface method.
2. Every public method.
3. Every internal method.
4. Every private method.
5. Every constructor.
6. Every property.
7. Every enum and enum member.
8. Every record and record parameter.
9. Every exception class.
10. Every delegate and event.
11. Every CLI command and option.
12. Every MCP tool and argument.
13. Every README and package description.

Every method gets a `<summary>`.

Every public method gets full parameter, return and exception documentation.

Do not use `<inheritdoc/>` as the only documentation.

Write an explicit summary on every method.

### 7.2 ASD-STE100 rules

1. Use one idea in each sentence.
2. Do not use more than 20 words in a sentence.
3. Use the active voice.
4. Use the present tense.
5. Use one word for one meaning.
6. Use "device" for the FDMS device.
7. Use "fiscal day" for the FDMS day.
8. Use "receipt" for a fiscal receipt.
9. Use "request" and "response" for HTTP messages.
10. Use "cancel" for cancellation.
11. Use "start" and "stop" for lifecycle actions.
12. Use "must" for a requirement.
13. Use "can" for a capability.
14. Use "may" for permission.
15. Do not use "should".
16. Do not use "will".
17. Do not use contractions.
18. Do not use idioms.
19. Do not use slang.
20. Do not use "e.g.", "i.e.", "etc." or "via".
21. Use "for example" and "that is".
22. Use "to" for ranges.
23. Use short paragraphs.
24. Use a list for steps.
25. Put a warning before a dangerous step.
26. Keep code names unchanged.
27. Use `<see cref="..."/>` for type references.
28. Use `<paramref name="..."/>` for parameter references.
29. Use `<c>...</c>` for code names.
30. End each sentence with a period.

### 7.3 Em dash ban

Do not use the em dash character U+2014.

Do not use these forms:

- U+2014
- `&mdash;`
- `&#8212;`
- `&#x2014;`

Use a period, comma, colon or parentheses instead.

Use "to" for ranges.

Do not use the en dash.

Apply this rule to these files:

- C# source files.
- XML documentation.
- Markdown files.
- NuGet descriptions.
- CLI help text.
- MCP tool descriptions.
- Error messages.
- Code comments.
- Release notes.
- Documentation HTML.

The current `README.md`, `CHANGELOG.md` and `docs/` pages contain em dashes.

Add a cleanup task for them in Phase 7.

### 7.4 XML documentation template

Use this template for a method:

```csharp
/// <summary>
/// Gets the device status.
/// </summary>
/// <param name="cancellationToken">
/// A token to cancel the operation.
/// </param>
/// <returns>
/// A task that contains the device status.
/// </returns>
/// <exception cref="FdmsApiException">
/// FDMS rejects the request.
/// </exception>
/// <exception cref="TransportException">
/// The network request fails.
/// </exception>
/// <exception cref="OperationCanceledException">
/// The caller cancels the operation.
/// </exception>
Task<GetStatusResponse> GetStatusAsync(
    CancellationToken cancellationToken = default);
```

Use this template for a dangerous method:

```csharp
/// <summary>
/// Signs and closes the open fiscal day.
/// </summary>
/// <param name="options">
/// The close options. The default is null.
/// </param>
/// <param name="cancellationToken">
/// A token to cancel the operation.
/// </param>
/// <returns>
/// A task that contains the close response.
/// </returns>
/// <remarks>
/// Warning: FDMS can reject the close after it accepts the request.
/// Poll GetStatusAsync until the day is closed or failed.
/// A day with a Red validation error cannot be closed by the device.
/// Use CloseDayOptions.Force only when ZIMRA tells you to do so.
/// </remarks>
/// <exception cref="DayNotClosableException">
/// The day has a receipt with a Red validation error.
/// </exception>
/// <exception cref="PendingSubmitException">
/// A receipt submit has an unknown result.
/// </exception>
Task<CloseDayResponse> CloseAsync(
    CloseDayOptions? options = null,
    CancellationToken cancellationToken = default);
```

Use this template for a constructor:

```csharp
/// <summary>
/// Creates a receipt service.
/// </summary>
/// <param name="api">The FDMS API client.</param>
/// <param name="state">The device state store.</param>
/// <param name="signer">The device signer.</param>
/// <param name="clock">The device clock.</param>
/// <param name="configCache">The device configuration cache.</param>
/// <param name="gate">The device operation gate.</param>
public ReceiptService(
    IFdmsApi api,
    IDeviceState state,
    ISigner signer,
    IClock clock,
    IDeviceConfigCache configCache,
    IDeviceOperationGate gate)
{
}
```

Use this template for a property:

```csharp
/// <summary>
/// Gets the number of receipts that wait for submission.
/// </summary>
int Count { get; }
```

Use this template for an enum member:

```csharp
/// <summary>
/// The fiscal day is open.
/// </summary>
FiscalDayOpened
```

### 7.5 Standard parameter text

Use these exact texts where they apply:

- `cancellationToken`: "A token to cancel the operation."
- `identity`: "The FDMS device identity."
- `input`: "The receipt data."
- `prepared`: "The prepared receipt."
- `options`: "The operation options. The default is null."
- `signer`: "The signer for the request. The default is null."
- `activationKey`: "The 8-character activation key from the FDMS portal."
- `state`: "The saved fiscal day state."
- `key`: "The storage key."
- `value`: "The value to save."
- `record`: "The journal record."
- `cursor`: "The journal cursor."
- `request`: "The transport request."

Do not use different words for these items.

### 7.6 Method summary matrix

Use these summaries. Adapt the parameter and return text to each signature.

#### IDeviceInfo

| Method | Summary |
| --- | --- |
| `GetConfigAsync` | Gets the device configuration and tax table. |
| `GetStatusAsync` | Gets the fiscal day and device status. |
| `PingAsync` | Checks the connection to FDMS. |

#### IFiscalDay

| Method | Summary |
| --- | --- |
| `OpenAsync` | Opens a fiscal day and saves the day state. |
| `CloseAsync` | Signs and closes the open fiscal day. |

#### IReceipts

| Method | Summary |
| --- | --- |
| `PrepareAsync` | Numbers, hashes and signs a receipt. Does not submit the receipt. |
| `SubmitAsync` | Numbers, hashes, signs and submits a receipt. |
| `SubmitPreparedAsync` | Submits a prepared receipt. |
| `GetQrData` | Returns the QR data for a prepared receipt. |

#### ICertificates

| Method | Summary |
| --- | --- |
| `RenewAsync` | Requests a new device certificate. |

#### IDeviceState

| Method | Summary |
| --- | --- |
| `Snapshot` | Gets a copy of the current fiscal day state. |
| `Restore` | Replaces the current fiscal day state with a saved state. |
| `GetLastIssuedGlobalNoAsync` | Gets the highest receipt global number that this device issued. |
| `GetPendingReceiptAsync` | Gets the receipt with an unknown submit result. |
| `DiscardPendingAsync` | Deletes the pending receipt marker. |
| `ClearAsync` | Deletes the saved fiscal day state. |

#### IReconciler

| Method | Summary |
| --- | --- |
| `ReconcileAsync` | Settles a pending receipt and loads the saved day state. |

#### IDeviceRegistrar

| Method | Summary |
| --- | --- |
| `RegisterAsync` | Registers a device and returns the issued certificate. |

#### IOfflineReceiptQueue

| Method | Summary |
| --- | --- |
| `Count` | Gets the number of receipts that wait for submission. |
| `OldestPendingAge` | Gets the age of the oldest pending receipt. |
| `EnqueueAsync` | Signs a receipt and adds it to the queue. |
| `SubmitOrEnqueueAsync` | Submits a receipt. Adds it to the queue when the network fails. |
| `FlushAsync` | Submits queued receipts in order. |

#### ISigner

| Method | Summary |
| --- | --- |
| `SignAsync` | Signs data with the device private key. |
| `GetPublicKeySpkiAsync` | Gets the public key in DER SubjectPublicKeyInfo format. |

#### IFdmsTransport

| Method | Summary |
| --- | --- |
| `SendAsync` | Sends one HTTP request to FDMS. |

#### IStorage

| Method | Summary |
| --- | --- |
| `GetAsync` | Gets a value by key. |
| `SetAsync` | Saves a value by key. |
| `DeleteAsync` | Deletes a value by key. |

#### IJournal

| Method | Summary |
| --- | --- |
| `AppendAsync` | Adds one record to the journal. |
| `ReadFromAsync` | Gets journal records from a cursor. |
| `CommitAsync` | Moves the journal commit cursor. |
| `GetCommittedAsync` | Gets the current journal commit cursor. |
| `ReplaceAsync` | Replaces one journal record. |

#### IDeviceConfigCache

| Method | Summary |
| --- | --- |
| `Current` | Gets the last device configuration. |
| `Set` | Saves the device configuration. |

#### IDeviceOperationGate

| Method | Summary |
| --- | --- |
| `AcquireAsync` | Waits for exclusive access to the device state. |

#### IRetryDelay

| Method | Summary |
| --- | --- |
| `DelayAsync` | Waits before the next retry. |

#### IRandom

| Method | Summary |
| --- | --- |
| `NextDouble` | Returns a random number from 0.0 to 1.0. |

#### ICsrBuilder

| Method | Summary |
| --- | --- |
| `BuildAsync` | Builds and signs a PKCS#10 certificate request. |

### 7.7 Exception and return rules

1. Document every exception that the method can throw.
2. Do not document `OperationCanceledException` for synchronous methods.
3. Document `OperationCanceledException` for every async method.
4. Document `null` results.
5. Document empty list results.
6. Document partial success.
7. Document asynchronous behavior.
8. Document retry behavior.
9. Document persistence side effects.
10. Document state changes.
11. Document warnings before dangerous actions.
12. Keep `<summary>` to one sentence.
13. Put detail in `<remarks>`.
14. Do not put a warning in `<summary>`.

### 7.8 Documentation enforcement

Add these build settings:

- `GenerateDocumentationFile=true`.
- `TreatWarningsAsErrors=true`.
- `CS1591` as an error for missing XML comments.
- `CS1573` as an error for a missing parameter comment.
- `CS1574` as an error for an invalid `cref`.
- `CS1587` as an error for a misplaced XML comment.
- `SA1600` as an error for missing documentation.
- `SA1611` as an error for missing parameter documentation.
- `SA1615` as an error for missing return documentation.
- `SA1618` as an error for missing type parameter documentation.
- `SA1623` as an error for a property summary.
- `SA1629` as an error for a missing period.
- `SA1642` as an error for a constructor summary.

Add a test named `DocumentationStyleTests`.

The test must check these rules:

1. Every C# file contains no U+2014.
2. Every Markdown file contains no U+2014.
3. Every XML documentation block contains no `&mdash;`.
4. Every XML documentation block contains no `&#8212;`.
5. Every public method has a `<summary>`.
6. Every public method has a `<returns>` when it returns a value.
7. Every public method has `<param>` for each parameter.
8. Every public method has `<exception>` for each documented exception.
9. Every dangerous method has a `Warning:` in `<remarks>`.
10. Every sentence in XML documentation is 20 words or fewer.

Add `dotnet format --verify-no-changes` to CI.

Add `dotnet build /warnaserror` to CI.

Add a documentation review checklist to the pull request template.

## 8. Testability

### 8.1 Testability rules

1. Every service has a public interface.
2. Every service uses constructor injection.
3. Every dependency is an interface or a pure function.
4. No service uses `DateTime.Now`, `DateTimeOffset.UtcNow`, `Task.Delay`, `Random.Shared`, `new HttpClient`, `File` or static mutable state.
5. Use `IClock` for time.
6. Use `IRetryDelay` for retries.
7. Use `IFdmsApi` for FDMS calls.
8. Use `IDeviceOperationGate` for concurrency.
9. Use `IStorage` and `IJournal` for persistence.
10. Use `ISigner` and `ICsBuilder` for crypto.
11. Every service has a public constructor.
12. Do not use a service locator inside a service.
13. Do not use `IServiceProvider` inside a service.
14. Test pure functions directly.
15. Test adapters with real temp directories or stub handlers.
16. Publish `Zimra.Fdms.Testing` with fakes and builders.

### 8.2 Test seams

| Interface | Used by | Purpose | Test fake |
| --- | --- | --- | --- |
| `IFdmsApi` | All device services | Typed FDMS calls | `FakeFdmsApi` |
| `IFdmsTransport` | `FdmsClient` | Raw HTTP | `FakeTransport` |
| `ISigner` | `IReceipts`, `IFiscalDay`, `ICertificates`, `IDeviceRegistrar` | Signing | `FakeSigner` |
| `ICsBuilder` | `ICertificates`, `IDeviceRegistrar` | CSR creation | `FakeCsrBuilder` |
| `IClock` | `IFiscalDay`, `IReceipts`, `IOfflineReceiptQueue`, `DeviceStateStore` | Time | `FixedClock` |
| `IRetryDelay` | `FdmsClient` | Retry backoff | `NoRetryDelay` |
| `IRandom` | `JitteredRetryDelay` | Jitter | `FixedRandom` |
| `IDeviceOperationGate` | `IFiscalDay`, `IReceipts`, `IReconciler` | Serialize state changes | `FakeGate` |
| `IDeviceState` | `IFiscalDay`, `IReceipts`, `IReconciler`, `IOfflineReceiptQueue` | State and pending marker | `FakeDeviceState` |
| `IDeviceConfigCache` | `IDeviceInfo`, `IReceipts` | QR URL cache | `InMemoryConfigCache` |
| `IStorage` | `DeviceStateStore` | Key-value persistence | `MemoryStorage` |
| `IJournal` | `IOfflineReceiptQueue` | Journal | `RecordingJournal` |
| `HttpMessageHandler` | `MtlsHttpTransport` | HTTP stubbing | `StubHttpMessageHandler` |

Do not make services depend on `FdmsClient` directly.

Make them depend on `IFdmsApi`.

Do not make services depend on `DeviceStateStore` directly.

Make them depend on `IDeviceState`.

### 8.3 Service constructors

Make every service constructor public.

Use only interfaces and pure helpers.

```csharp
public sealed class DeviceInfo : IDeviceInfo
{
    public DeviceInfo(
        IFdmsApi api,
        IDeviceConfigCache configCache)
    {
    }
}

public sealed class FiscalDayService : IFiscalDay
{
    public FiscalDayService(
        IFdmsApi api,
        IDeviceState state,
        ISigner signer,
        IClock clock,
        IDeviceOperationGate gate)
    {
    }
}

public sealed class ReceiptService : IReceipts
{
    public ReceiptService(
        IFdmsApi api,
        IDeviceState state,
        ISigner signer,
        IClock clock,
        IDeviceConfigCache configCache,
        IDeviceOperationGate gate)
    {
    }
}

public sealed class CertificateService : ICertificates
{
    public CertificateService(
        IFdmsApi api,
        ISigner signer,
        ICsrBuilder csrBuilder)
    {
    }
}

public sealed class Reconciler : IReconciler
{
    public Reconciler(
        IFdmsApi api,
        IDeviceState state,
        IDeviceOperationGate gate)
    {
    }
}

public sealed class OfflineReceiptQueue : IOfflineReceiptQueue
{
    public OfflineReceiptQueue(
        IReceipts receipts,
        IDeviceState state,
        IJournal journal,
        IClock clock)
    {
    }
}

public sealed class DeviceRegistrar : IDeviceRegistrar
{
    public DeviceRegistrar(
        IFdmsApi api,
        ISigner signer,
        ICsrBuilder csrBuilder)
    {
    }
}
```

Keep `FdmsClient` internal.

Test it with `InternalsVisibleTo`.

Keep `DeviceStateStore` internal or public. Prefer internal with `InternalsVisibleTo`.

### 8.4 Testing package

Add `Zimra.Fdms.Testing`.

It contains these fakes:

- `FakeFdmsApi`
- `FakeTransport`
- `FakeSigner`
- `FakeCsrBuilder`
- `FixedClock`
- `NoRetryDelay`
- `FixedRandom`
- `FakeGate`
- `FakeDeviceState`
- `InMemoryConfigCache`
- `RecordingJournal`
- `StubHttpMessageHandler`
- `TestServiceBuilder`

`TestServiceBuilder` wires the real DI registration and replaces chosen services with fakes.

```csharp
var services = new ServiceCollection();

services.AddZimraFdmsForTesting(
    identity,
    options,
    new FdmsDeviceDependencies
    {
        Signer = fakeSigner,
        Transport = fakeTransport
    });

services.ReplaceSingleton<IFdmsApi>(fakeApi);
services.ReplaceSingleton<IClock>(fixedClock);
services.ReplaceSingleton<IDeviceState>(fakeState);

await using var provider = services.BuildServiceProvider();

var receipts = provider.GetRequiredService<IReceipts>();
```

Do not use a mocking library.

Hand-written fakes are deterministic and fast.

### 8.5 Test project layout

```text
tests/
  Zimra.Fdms.Tests/
    Contract/
      CancellationTokenContractTests.cs
      PublicApiContractTests.cs
      JsonContractTests.cs
      DocumentationStyleTests.cs
    Unit/
      Signing/
      Money/
      Timestamp/
      Qr/
      Errors/
      Clock/
      RetryDelay/
    Services/
      DeviceInfoTests.cs
      FiscalDayServiceTests.cs
      ReceiptServiceTests.cs
      CertificateServiceTests.cs
      ReconcilerTests.cs
      OfflineReceiptQueueTests.cs
      DeviceStateStoreTests.cs
      DeviceRegistrarTests.cs
      DependencyInjectionTests.cs
    Adapters/
      FdmsClientTests.cs
      MtlsHttpTransportTests.cs
      FileStorageTests.cs
      FileJournalTests.cs
      ProfileStoreTests.cs
    Crash/
      PendingMarkerTests.cs
      ReconcileCrashTests.cs
    Integration/
      SimulatorHttpTests.cs
  Zimra.Fdms.Simulator.Tests/
  Zimra.Fdms.Cli.Tests/
  Zimra.Fdms.Mcp.Tests/
  Zimra.Fdms.Conformance.Tests/
```

### 8.6 Service test matrix

| Service | Required tests |
| --- | --- |
| `DeviceInfo` | Config cache update, status, ping, API error, cancellation |
| `FiscalDayService` | Open, close, close with Red errors, close with force, API error, gate, cancellation |
| `ReceiptService` | Prepare, submit, submit prepared, validation errors, pending marker order, QR, cancellation |
| `CertificateService` | Renew with same signer, renew with new signer, CSR failure, API error, cancellation |
| `Reconciler` | None, confirmed, resubmitted, rejected, network failure, gate, cancellation |
| `OfflineReceiptQueue` | Enqueue, submit or enqueue, FIFO flush, stop on rejection, oldest age, cancellation |
| `DeviceStateStore` | Snapshot copy, restore copy, last global monotonic, pending marker, clear, device mismatch |
| `DeviceRegistrar` | Success, wrong activation key, CSR failure, cancellation |
| `FdmsClient` | Paths, headers, JSON body, problem details, date header, retries, cancellation |
| `MtlsHttpTransport` | Headers, body, status, transport error, timeout, cancellation |
| `ProfileStore` | Missing profile, incomplete profile, device mismatch, day state, last global, date validation |

### 8.7 Concurrency tests

1. Call `IReceipts.SubmitAsync` twice at the same time.
2. Assert the gate serializes the calls.
3. Assert only one pending marker exists.
4. Call `IFiscalDay.CloseAsync` while `IReceipts.SubmitAsync` runs.
5. Assert no state corruption.
6. Call `IReconciler.ReconcileAsync` twice at the same time.
7. Assert one reconcile action.

### 8.8 Build-time enforcement

Add these analyzers and rules:

- `Microsoft.CodeAnalysis.NetAnalyzers`
- `Microsoft.CodeAnalysis.PublicApiAnalyzers`
- `Microsoft.CodeAnalysis.BannedApiAnalyzers`
- `CA2016`: forward the cancellation token.
- `CA2007`: use `ConfigureAwait(false)`.
- `CA1001`: dispose owned resources.
- `CA1063`: implement dispose correctly.

Ban these symbols in service code:

- `System.DateTime.Now`
- `System.DateTimeOffset.UtcNow`
- `System.Random.Shared`
- `System.Threading.Tasks.Task.Delay(int)`
- `System.Net.Http.HttpClient..ctor`
- `System.IO.File`

Allow them only in adapters with a local suppression.

Add `PublicAPI.Shipped.txt` and `PublicAPI.Unshipped.txt` for every package.

### 8.9 CI tests

Run these steps:

1. Build with warnings as errors.
2. Run unit tests.
3. Run service tests with coverage.
4. Run crash tests.
5. Run integration tests.
6. Run CLI tests.
7. Run MCP tests.
8. Run conformance both directions.
9. Upload coverage.
10. Fail if service coverage is below 90%.
11. Fail if total coverage is below 85%.
12. Fail if the public API changed without an update to `PublicAPI.Unshipped.txt`.
13. Fail if a file contains U+2014.
14. Fail if XML documentation is missing.

## 9. Implementation phases

### Phase 0: Scaffold and contracts

1. Create the solution and projects.
2. Add the service interfaces.
3. Add the dependency injection extension signature.
4. Add the CancellationToken reflection test.
5. Add the documentation style guide.
6. Add the XML templates.
7. Add the analyzer rules.
8. Add the documentation tests.
9. Add CI.

Exit: the contracts compile. The reflection test passes on the empty service set.

### Phase 1: Primitives and signing

1. Port models and enums.
2. Port money, timestamps and invariant formatting.
3. Port hashing, base64, PEM and DER helpers.
4. Port code point ordering with `Rune`.
5. Port receipt and fiscal day canonical strings.
6. Port tax computation and counter accumulation.
7. Port QR generation.
8. Port the error catalogue.
9. Implement the conformance responder.
10. Port the signing and vector tests.

Exit: all vectors pass against the C# responder. The JS runner accepts it.

### Phase 2: Services

Build these in order:

1. `DeviceStateStore`
2. `DeviceInfo`
3. `FiscalDayService`
4. `ReceiptService`
5. `CertificateService`
6. `Reconciler`
7. `OfflineReceiptQueue`
8. `DeviceRegistrar`
9. Dependency injection registration

Write one test class per service.

Write documentation for every method, constructor and interface.

Exit: a full fiscal day runs through the services. Crash tests pass. Cancellation tests pass.

### Phase 3: Profile and CLI

1. Implement the profile store.
2. Implement CLI commands.
3. Resolve services from the container.
4. Pass the CLI cancellation token to every service call.
5. Port CLI tests.
6. Write documentation for every command and option.
7. Use ASD-STE100 in CLI help output.

Exit: CLI tests pass. `Ctrl+C` cancels cleanly.

### Phase 4: Simulator

1. Implement the simulator CA.
2. Generate the root and server certificates.
3. Implement CSR issuance and CSR signature verification.
4. Implement Kestrel HTTPS with client certificate support.
5. Implement Public and Device routes.
6. Implement receipt validation codes.
7. Implement asynchronous CloseDay.
8. Implement the GetStatus latest date quirk.
9. Implement faults.
10. Implement the simulator CLI.
11. Port simulator and CLI simulator tests.

Exit: full day over real mutual TLS passes. All validation codes and faults pass.

### Phase 5: MCP server

1. Add `ModelContextProtocol` 2.2.x.
2. Create the stdio server.
3. Register the eight tools with the same names and arguments.
4. Keep tool annotations.
5. Return errors in-band.
6. Support a default profile and a per call profile.
7. Port MCP tests.
8. Write documentation for every tool and argument.
9. Use ASD-STE100 in tool descriptions.

Exit: modern and legacy clients both connect. Tool list and sample receipt match.

### Phase 6: Conformance in both directions

1. Implement the C# conformance runner.
2. Keep the C# responder.
3. Add a CI step. The JS runner calls the C# responder.
4. Add a CI step. The C# runner calls the JS reference responder.
5. Add HTTP golden fixtures from the TypeScript SDK.

Exit: both conformance directions pass.

### Phase 7: Documentation and packaging

1. Write `dotnet/README.md`.
2. Add quick start, amounts, crash safety, offline queue, renewal, CLI, MCP and simulator sections.
3. Add XML documentation for all public APIs.
4. Add samples.
5. Add package metadata, SourceLink and symbols.
6. Run `dotnet pack`.
7. Install the package in a clean sample project.
8. Add the NuGet publish workflow.
9. Remove em dashes from `README.md`.
10. Remove em dashes from `CHANGELOG.md`.
11. Remove em dashes from `docs/`.
12. Run the documentation style test on the full repository.

Exit: the package installs and the sample runs. The documentation test passes.

## 10. Packaging and release

- Add `.github/workflows/dotnet-ci.yml`.
- Run restore, build, test and pack on push and pull request.
- Run the conformance cross tests in CI.
- Add `.github/workflows/dotnet-publish.yml`.
- Trigger on `dotnet-v*` tags.
- Push to NuGet with `NUGET_API_KEY`.
- Publish symbols and SourceLink.
- Add package signing later if required.
- Start with version `0.1.0-preview.1`.
- Move to `1.0.0` after conformance, simulator and CLI tests pass.

Use these package settings:

- `PackageId`, `Version`, `Authors`, `Description`.
- `PackageLicenseExpression=MIT`.
- `PackageProjectUrl`, `RepositoryUrl`, `RepositoryType=git`.
- `PackageReadmeFile`, `PackageIcon`, `PackageTags`.
- `GenerateDocumentationFile=true`.
- `PublishRepositoryUrl=true`, `EmbedUntrackedSources=true`.
- `IncludeSymbols=true`, `SymbolPackageFormat=snupkg`.
- `ContinuousIntegrationBuild=true`.

## 11. Risks and mitigations

| Risk | Mitigation |
| --- | --- |
| Canonical string mismatch | Make vectors the first test. Run both conformance directions in CI. |
| ECDSA DER and P1363 mismatch | Use `Rfc3279DerSequence`. Test both formats. |
| Money rounding differences | Use `long` cents. Avoid `double`. Add edge case tests. |
| Date and timezone differences | Use `DateTimeOffset`, local wire formatting and invariant culture. Test fixed dates. |
| Tax code sort differences | Compare Unicode code points with `Rune`. Test astral characters. |
| JSON null and omitted field differences | Use explicit attributes and golden request tests. |
| Windows mTLS certificate loading | Reimport as PKCS#12. Test on Windows and Linux. |
| Kestrel client certificate validation | Test valid, missing and wrong device certificates. |
| Simulator CSR verification | Use `CertificateRequest.LoadSigningRequest`. Add a corrupted CSR test. |
| MCP protocol changes | Use the official C# SDK. Pin the version. Test modern and legacy clients. |
| Concurrent profile use | Add a profile lock. Test two processes. |
| Scope is large | Deliver in phases. Keep the TypeScript SDK as the reference. |
| Documentation drift | Add analyzers, style tests and a pull request checklist. |

## 12. Open decisions

1. Confirm the first version. Use `0.1.0-preview.1` or `0.5.0`.
2. Confirm the tag scheme. Use `dotnet-v*` or a shared `v*`.
3. Confirm whether to publish `Zimra.Fdms.Mcp` as a separate package.
4. Confirm whether NuGet package signing is required.
5. Confirm whether profile files must interoperate with the TypeScript CLI. The plan assumes no.
6. Confirm whether .NET 8 stays after its support ends in November 2026.
7. Confirm whether `Zimra.Fdms.Testing` is public.

## 13. Definition of done

- No public `FiscalDevice` class exists.
- Every service has an interface and a public constructor.
- Every dependency is injected through an interface.
- Every async public and internal method accepts `CancellationToken cancellationToken = default`.
- No method uses the parameter name `ct`.
- The reflection test finds no async method without a `CancellationToken`.
- Every method has a `<summary>`.
- Every public method has full `<param>`, `<returns>` and `<exception>` blocks.
- Every constructor has a summary and parameter documentation.
- Every property has a summary.
- Every enum member has a summary.
- Every dangerous method has a warning.
- No file contains U+2014.
- No file contains `&mdash;` or `&#8212;`.
- All documentation uses ASD-STE100.
- All XML documentation sentences are 20 words or fewer.
- Documentation analyzers pass with warnings as errors.
- The documentation style test passes.
- Every service has a dedicated test class with fakes.
- No service uses ambient time, randomness, HTTP, file I/O or static state.
- `Zimra.Fdms.Testing` provides a fake for every dependency.
- Cancellation is tested in every service.
- Concurrency is tested for day, receipt and reconcile operations.
- Banned API analyzer is clean.
- Public API analyzer is clean.
- Service coverage is at least 90%.
- Total coverage is at least 85%.
- All published vectors pass against the C# responder.
- The C# runner passes against the JS reference responder.
- The simulator passes the full mutual TLS test suite.
- The CLI completes register, open, submit and close against the simulator.
- The MCP server exposes the same eight tools.
- Package installation and the sample app work on a clean machine.
- CI is green on Windows, Ubuntu and macOS for .NET 8 and .NET 10.
- The package has no high or critical vulnerabilities.
- The root README links to the C# package and documentation.
