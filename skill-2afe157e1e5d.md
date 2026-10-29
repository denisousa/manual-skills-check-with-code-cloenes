<!-- YosaRama/hermes-backup:skills/software-development/dotnet-xunit-testing/SKILL.md | https://github.com/YosaRama/hermes-backup/blob/HEAD/skills/software-development/dotnet-xunit-testing/SKILL.md -->

---
name: dotnet-xunit-testing
description: "Create and maintain xUnit test projects for .NET (ASP.NET Core). Covers project setup, Moq mocking patterns, coverage with coverlet + SonarQube, and common pitfalls."
version: 1.0.0
author: Hermes Agent
license: MIT
metadata:
  hermes:
    tags: [dotnet, xunit, testing, moq, coverlet, sonarqube, csharp]
    related_skills: [writing-plans, test-driven-development, subagent-driven-development]
---

# .NET xUnit Testing

## Overview

Create and maintain comprehensive xUnit test projects for .NET / ASP.NET Core applications. Covers project scaffolding, mocking strategies for common .NET abstractions (IConfiguration, HttpClient, Redis, EF Core), coverage tooling with SonarQube integration, and SDK version management.

## When to Load

- User asks for unit tests, xUnit tests, or test coverage on a .NET project
- Setting up a new test project alongside an existing .NET solution
- Configuring SonarQube or coverlet for a .NET codebase
- IConfiguration or HttpClient mocking is needed
- User says "add tests" or "write tests" in a .NET repo

## Test Project Setup

### csproj Template

Target framework MUST match the main project. Always verify with `dotnet --list-sdks` first.

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>  <!-- Match main project -->
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <IsPackable>false</IsPackable>
    <IsTestProject>true</IsTestProject>

    <!-- Coverlet for SonarQube-compatible coverage -->
    <CollectCoverage>true</CollectCoverage>
    <CoverletOutputFormat>opencover,cobertura</CoverletOutputFormat>
    <CoverletOutput>./TestResults/coverage</CoverletOutput>
    <ExcludeByAttribute>GeneratedCodeAttribute</ExcludeByAttribute>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="coverlet.collector" Version="6.0.4">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
    </PackageReference>
    <PackageReference Include="coverlet.msbuild" Version="6.0.4">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
    </PackageReference>
    <PackageReference Include="FluentAssertions" Version="7.2.0" />
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.13.0" />
    <PackageReference Include="Moq" Version="4.20.72" />
    <PackageReference Include="xunit" Version="2.9.3" />
    <PackageReference Include="xunit.runner.visualstudio" Version="3.0.2">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
    </PackageReference>
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\Main.Api\Main.Api.csproj" />
    <ProjectReference Include="..\Main.Core\Main.Core.csproj" />
  </ItemGroup>
</Project>
```

### GlobalUsings.cs

Create a `GlobalUsings.cs` in the test project root:

```csharp
global using Xunit;
global using Moq;
global using FluentAssertions;
global using Microsoft.Extensions.Configuration;
global using Microsoft.AspNetCore.Mvc;
global using Senoko.Core.Models.ApiResponse;  // Project-specific
```

### Add to solution

```bash
cd SolutionDir
dotnet sln Solution.sln add ../Tests.Project/Tests.Project.csproj
```

## Mocking Patterns

### IConfiguration — Use InMemoryCollection (NOT Mock<IConfigurationSection>)

**PITFALL:** Mocking `IConfigurationSection` with `GetChildren()` and `Get<T>()` for the configuration binder almost never works. The binder uses internal recursion that Moq can't replicate. Always use `ConfigurationBuilder().AddInMemoryCollection()` instead.

**RIGHT way:**

```csharp
private static IConfiguration CreateConfiguration()
{
    var settings = new Dictionary<string, string?>
    {
        { "Jwt:Key", "test-key-with-sufficient-length" },
        { "Jwt:Issuer", "TestIssuer" },
        { "Jwt:Audience", "TestAudience" },
        { "JwtTokenExpiryInMinutes", "15" }
    };
    return new ConfigurationBuilder()
        .AddInMemoryCollection(settings)
        .Build();
}
```

**WRONG way (will fail on Get<T>()):**

```csharp
// DON'T DO THIS — Get<T>() binder won't resolve from mocked sections
var mockSection = new Mock<IConfigurationSection>();
mockSection.Setup(s => s.GetChildren()).Returns(...);
config.Setup(c => c.GetSection("Jwt")).Returns(mockSection.Object);
```

### HttpClient — Use DelegatingHandler stub

For services that depend on `HttpClient` (via `IHttpClientFactory` or direct injection), create a test handler:

```csharp
private class FakeHttpMessageHandler : DelegatingHandler
{
    private readonly HttpResponseMessage _response;
    public FakeHttpMessageHandler(HttpResponseMessage response) => _response = response;
    protected override Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken ct)
        => Task.FromResult(_response);
}

// Usage:
var handler = new FakeHttpMessageHandler(new HttpResponseMessage
{
    StatusCode = HttpStatusCode.OK,
    Content = new StringContent("{\"isSuccess\":true,\"result\":{...}}")
});
var httpClient = new HttpClient(handler);
```

### Redis / StackExchange.Redis — Mock IConnectionMultiplexer + IDatabase

```csharp
var dbMock = new Mock<IDatabase>();
dbMock.Setup(d => d.StringSetAsync(It.IsAny<RedisKey>(), It.IsAny<RedisValue>(),
    It.IsAny<Expiration>(), It.IsAny<ValueCondition>(), It.IsAny<CommandFlags>()))
    .ReturnsAsync(true);
dbMock.Setup(d => d.StringGetAsync(It.IsAny<RedisKey>(), It.IsAny<CommandFlags>()))
    .ReturnsAsync(new RedisValue("cached-value"));

var redisMock = new Mock<IConnectionMultiplexer>();
redisMock.Setup(r => r.GetDatabase(It.IsAny<int>(), It.IsAny<object>()))
    .Returns(dbMock.Object);
```

**PITFALL:** StackExchange.Redis `StringSetAsync` has multiple overloads. Moq may resolve to one with `Expiration` and `ValueCondition` parameters (not `TimeSpan?` and `When`). Use `It.IsAny<Expiration>()` and `It.IsAny<ValueCondition>()` to match any overload. If the API changes across versions, check the exact overload signature.

### EF Core DbContext — Use InMemory provider

```csharp
var options = new DbContextOptionsBuilder<AppDbContext>()
    .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())
    .Options;
var context = new AppDbContext(options);
```

Always use `Guid.NewGuid()` as the database name to isolate tests from each other.

**For repository testing**, instantiate the concrete repository with the InMemory context directly:

```csharp
public class CustomerExtraDetailsRepositoryTests : IDisposable
{
    private readonly AppDbContext _context;
    private readonly CustomerExtraDetailsRepository _repository;

    public CustomerExtraDetailsRepositoryTests()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseInMemoryDatabase($"TestDb_{Guid.NewGuid()}")
            .Options;
        _context = new AppDbContext(options);
        _repository = new CustomerExtraDetailsRepository(_context);
    }

    public void Dispose() => _context.Dispose();

    [Fact]
    public async Task Lock_NewCustomer_CreatesAndLocks()
    {
        var result = await _repository.Lock("customer-1");
        result.Should().BeTrue();

        var saved = await _context.Set<CustomerExtraDetails>()
            .FirstOrDefaultAsync(x => x.CustomerId == "customer-1");
        saved.Should().NotBeNull();
        saved!.IsLock.Should().BeTrue();
    }
}
```

**Upsert pattern (create-or-update):** Many repositories implement an upsert pattern — look up existing entity, update if found, create if not. Test both branches:

```csharp
// Branch 1: New entity (create)
await _repository.SetDefaultPremiseId("new-customer", 10);
// Verify entity was created with correct values

// Branch 2: Existing entity (update)
await _repository.SetDefaultPremiseId("existing-customer", 20);
// Verify DefaultPremiseId was updated from 10 → 20
```

**IMPORTANT:** The Infrastructure project must be referenced in the test csproj to access repository classes:

```xml
<ProjectReference Include="..\Senoko.Infrastructure\Senoko.Infrastructure.csproj" />
```

And `Microsoft.EntityFrameworkCore.InMemory` package must be added:

```xml
<PackageReference Include="Microsoft.EntityFrameworkCore.InMemory" Version="10.0.9" />
```

## Controller Testing

Controllers are tested by direct instantiation with mocked dependencies (constructor injection). No need for `WebApplicationFactory` unless testing middleware pipeline.

**IMPORTANT:** When testing async controller actions, always use `async Task` test methods with `await`. NEVER use `.Result` or `.Wait()` — these block the thread and SonarQube rule S4462 will flag them.

```csharp
// RIGHT: async test method
[Fact]
public async Task Get_ReturnsOk()
{
    var mockService = new Mock<IMyService>();
    mockService.Setup(s => s.GetData()).ReturnsAsync(expectedData);
    var controller = new MyController(mockService.Object);

    var result = await controller.Get();

    result.Should().BeOfType<OkObjectResult>();
}

// WRONG: blocking with .Result / .Wait()
[Fact]
public void Get_ReturnsOk()  // sync method
{
    var result = controller.Refresh(request).Result;   // BLOCKS — SonarQube S4462
    controller.Refresh(request).Wait();                 // BLOCKS — SonarQube S4462
}
```

## Coverage & SonarQube

### Local coverage (no SonarQube)

```bash
dotnet test Tests.Project/Tests.Project.csproj
# Coverage files auto-generated at Tests.Project/TestResults/
# - coverage.opencover.xml (for SonarQube)
# - coverage.cobertura.xml (for ReportGenerator/Cobertura viewers)
```

### Install SonarQube scanner

```bash
dotnet tool install --global dotnet-sonarscanner
```

### Full SonarQube pipeline (local CLI)

```bash
# Step 1: Begin analysis
dotnet sonarscanner begin \
  /k:"project-key" \
  /d:sonar.host.url="https://sonarqube.example.com" \
  /d:sonar.token="your-token" \
  /d:sonar.cs.opencover.reportsPaths="**/coverage.opencover.xml"

# Step 2: Build
dotnet build

# Step 3: Run tests with coverage
dotnet test Tests.Project/Tests.Project.csproj

# Step 4: Upload results
dotnet sonarscanner end /d:sonar.token="your-token"
```

### GitHub Actions CI Config

```yaml
jobs:
  build:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'

      - name: SonarQube Begin
        run: |
          dotnet-sonarscanner begin `
            /k:"$env:SONAR_PROJECT_KEY" `
            /d:sonar.token="$env:SONAR_TOKEN" `
            /d:sonar.host.url="$env:SONAR_HOST_URL" `
            /d:sonar.cs.opencover.reportsPaths="Tests.Project\TestResults\coverage.opencover.xml" `
            /d:sonar.coverage.exclusions="Tests.Project\**" `
            /d:sonar.cpd.exclusions="Tests.Project\**"

      - name: Build
        run: dotnet build

      - name: Test
        run: dotnet test Tests.Project/Tests.Project.csproj

      - name: SonarQube End
        run: dotnet-sonarscanner end /d:sonar.token="$env:SONAR_TOKEN"
```

**Critical CI rules:**

1. `begin` / build / test / `end` MUST be in the **same job** — the `.sonarqube/` folder is wiped between jobs.
2. On Windows runners, use **backslashes** in paths, not forward-slashes or globs (`**/coverage.xml` may fail).
3. Always exclude the test project from **coverage** (`sonar.coverage.exclusions`) and **duplication** (`sonar.cpd.exclusions`). Test files count as "new code" with 0% coverage if not excluded, failing quality gates.
4. Do NOT use `--no-build` with `dotnet test` when coverage is needed — it may skip coverlet instrumentation.
5. Add these to `.gitignore`: `TestResults/`, `*.opencover.xml`, `*.cobertura.xml`, `.sonarqube/`

### Quality Gate Exclusions

| Parameter | Why |
|-----------|-----|
| `sonar.coverage.exclusions` | Test files have 0% self-coverage; excluding them prevents "0% coverage on new code" gate failure |
| `sonar.cpd.exclusions` | Test files have repetitive patterns (Arrange-Act-Assert); excluding prevents "too much duplication" gate failure |

### Agent Workflow — NEVER Commit

When implementing tests (or any code), the agent MUST stage changes for review but **NEVER commit**. The user reviews and commits manually. Use `git add` to stage, then present the staged diff for the user to commit.

## .NET SDK Version Management

### Check installed SDKs

```bash
dotnet --list-sdks
```

### Install additional SDK

When the installed SDK doesn't support the project's target framework (e.g., .NET 8 SDK can't build net10.0), use the dotnet-install script:

```bash
curl -sSL https://dot.net/v1/dotnet-install.sh -o /tmp/dotnet-install.sh
chmod +x /tmp/dotnet-install.sh
/tmp/dotnet-install.sh --channel 10.0 --install-dir ~/.dotnet
```

The `--channel` flag accepts major.minor version (e.g., `10.0`, `9.0`, `8.0`). The script auto-detects the latest patch.

## Test Structure Conventions

### File organization

```
Tests.Project/
├── Helpers/           # Static helper classes (Encryption, DateTime, etc.)
├── Enums/             # Enum values and extension methods
├── Models/            # Records, DTOs, computed properties
├── Services/          # Business logic services (one file per service)
├── Controllers/       # API controllers (one file per controller)
└── Middleware/        # Exception handlers, pipeline middleware
```

### Test naming

```
MethodName_Scenario_ExpectedResult
```

Example: `GetOutstandingBillStatus_Overdue_ReturnsOVERDUE`

### Private mock factories

Each test file should have private static factory methods for creating mocks with default happy-path setups. Individual tests override specific setups as needed. This keeps tests DRY.

```csharp
private static Mock<IDashboardApiService> CreateDashboardApiMock(
    CustomerDetail? customerDetail = null)
{
    var mock = new Mock<IDashboardApiService>();
    // Default happy-path setup
    mock.Setup(s => s.GetCustomerBillingDetailAsync(It.IsAny<...>()))
        .ReturnsAsync(new ... { CustomerDetail = customerDetail ?? CreateDefault() });
    return mock;
}
```

### Assertions

Use FluentAssertions for all assertions:
- `result.Should().BeOfType<OkObjectResult>()` — type checks
- `result.Success.Should().BeTrue()` — boolean checks
- `result.Result!.Status.Should().Be("OVERDUE")` — value checks
- `mock.Verify(s => s.Method(It.Is<Type>(t => t.Prop == "val")), Times.Once)` — interaction verification

## Pitfalls

1. **Mock<IConfigurationSection> + Get<T>()**: The configuration binder uses internal `Bind()` recursion. Moq can't fake this. Use `ConfigurationBuilder().AddInMemoryCollection()`.

2. **`.Result` / `.Wait()` in test methods**: SonarQube rule S4462 flags blocking calls on Task. Always use `async Task` test methods with `await`. Never call `.Result` or `.Wait()` in tests.

3. **Required members in C# records/classes**: When a model has `required` properties, object initializers must set them or the constructor must be called with arguments. Check model definitions before writing test data builders.

4. **StackExchange.Redis overload ambiguity**: `StringSetAsync` has multiple overloads with different parameter types. Use `It.IsAny<Expiration>()` not `It.IsAny<TimeSpan?>()`.

5. **Non-nullable value types in models**: `DateTime` (not `DateTime?`) can't represent null. If production code uses null-conditional (`?.`) to read it, the null case can only be triggered by the parent object being null, not the value itself.

6. **Coverlet format**: SonarQube needs `opencover` format, not just `cobertura`. Always output both formats with `<CoverletOutputFormat>opencover,cobertura</CoverletOutputFormat>`.

7. **SonarQube quality gate failures for test projects**: When test files are new code in a PR, they show 0% coverage (tests don't cover themselves) and high duplication (similar test patterns). Always exclude the test project with `sonar.coverage.exclusions` and `sonar.cpd.exclusions` in the `begin` step.

8. **SonarQube duplication also hits repository classes**: CRUD repositories (BaseRepository, CustomerExtraDetailsRepository, PremiseExtraDetailsRepository, etc.) share inherent structural patterns (FindBy, Upsert, Lock/Unlock) that SonarQube flags as duplication. Exclude them too: `/d:sonar.cpd.exclusions="Tests\**,**\Repositories\**"`

9. **Windows path separator in SonarQube**: On Windows CI runners, use backslashes in `sonar.cs.opencover.reportsPaths` (e.g., `TestProject\\TestResults\\coverage.opencover.xml`). Glob patterns (`**/...`) with forward slashes may fail.

10. **`dotnet test --no-build` breaks coverlet**: The `--no-build` flag can skip coverlet instrumentation. When coverage is needed, use plain `dotnet test`.

11. **Claim types in JWT**: Use `System.Security.Claims.ClaimTypes.Role` (which is `"http://schemas.microsoft.com/ws/2008/06/identity/claims/role"`) not the string `"role"` when asserting JWT claims.

12. **Agent must NOT commit**: Stage changes with `git add` and present the diff. User reviews and commits manually.

13. **PatchAsync / JsonElement type conversion**: When testing `PatchAsync` or any method receiving `Dictionary<string, object?>` from ASP.NET Core JSON deserialization, values arrive as `JsonElement`, not raw C# types. Always pass `JsonElement` values in tests — passing raw `bool` fails `Convert.ChangeType(bool, Nullable<bool>)` inside `JsonHelpers.ConvertJsonElement`:

```csharp
// RIGHT: Use JsonElement (simulates real ASP.NET Core binding)
using var doc = JsonDocument.Parse("true");
var updates = new Dictionary<string, object?> { { "IsLock", doc.RootElement } };

// WRONG: Raw C# value fails Convert.ChangeType for nullable target types
var updates = new Dictionary<string, object?> { { "IsLock", true } };
```

14. **Whitespace vs empty string validation**: `string.IsNullOrEmpty("   ")` returns `false`. Methods using this check accept whitespace-only strings. Don't write tests expecting whitespace rejection unless `string.IsNullOrWhiteSpace` is used.

15. **`GC.SuppressFinalize(this)` in IDisposable test classes**: SonarQube requires `GC.SuppressFinalize(this)` in `Dispose()` methods. Test classes implementing `IDisposable` (e.g., for cleaning up EF Core InMemory contexts) must include it:

```csharp
public void Dispose()
{
    _context.Dispose();
    GC.SuppressFinalize(this);
}
```

16. **Controller class renamed after test creation**: When a controller is split or renamed (e.g., `CustomersController` → `CustomerController`), existing test files break with `CS0246: The type or namespace name 'XxxController' could not be found`. When this happens, check `git diff develop --name-only` for renamed/moved controller files, update the test class name and constructor call, and check for new dependencies (the renamed controller may have gained new constructor parameters).

17. **AppDbContext moved between projects**: When `AppDbContext` moves from `Senoko.Core.Data` to `Senoko.Infrastructure.Data`, all test files referencing it need their `using` directives updated. Check `using Senoko.Infrastructure.Data` vs `using Senoko.Core.Data` in repository tests. Also verify the DbContext includes the entity types needed by the tests — the Infrastructure context may not have all DbSets the Core one had.

## Commands Reference

```bash
# Run all tests
dotnet test Tests.Project/Tests.Project.csproj
...
```

## Reference Files

- `references/github-actions-governance-wrapper.md` — Governance wrapper pattern for converting self-contained CI workflows to org-level reusable workflow calls (Section-specific).
# Run specific test class
dotnet test Tests.Project/Tests.Project.csproj --filter "FullyQualifiedName~BillServiceTests"

# Run with verbose output
dotnet test Tests.Project/Tests.Project.csproj --verbosity normal

# Build only (faster than test for checking compilation)
dotnet build Tests.Project/Tests.Project.csproj

# Check installed SDKs
dotnet --list-sdks
```
