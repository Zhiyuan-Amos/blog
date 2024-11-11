---
title: Practical Tests for .NET
tags: [test]
readtime: true
---

## TestContainers

Integration Tests best ensure that features are working. So, use TestContainers to spin up real instances of dependencies. It is sometimes helpful to run these tests in a container, so it's necessary to setup [Docker-in-Docker](https://dotnet.testcontainers.org/examples/dind/).

Note that TestContainers can be initialized through static classes with static variables and static constructors, so the containers are started before any of the tests run.

### Implementation

The Dockerfile for running tests is similar to a typical Dockerfile. The main complexity is with the `ENTRYPOINT` command which looks like `["dotnet", "test", "{project}", "--logger", "xunit;LogFilePath={path};verbosity=detailed"]`. Then, run the following commands:

```
// linux
docker compose -f {file} up
docker compose -f {file} down --rmi all

// non-linux
docker compose -f {file} build
docker compose -f {file} run -e TESTCONTAINERS_HOST_OVERRIDE=host.docker.internal
docker compose -f {file} down --rmi all
```

Note to retrieve `Hostname` and `GetMappedPublicPort` from the WireMock library instead in order for DinD to work.

## WireMock

If it's not easy to spin up or initialize a real instance of the dependency, use WireMock to mock the expected responses.

To avoid shutting down and spinning up a new instance of WireMock server for each test, consider sending a `x-test-key` header and DI-ing a HttpClient that sends the header to dependencies. So, the WireMock server returns the corresponding response depending on the value of the `x-test-key` header. Can also evaluate the possibility of de-registering a mocked response.

## Infrastructure

The Infrastructure for Integration Tests generally involves the following:

1. Authorization. Generally, the real instance of Authorization Server isn't deployed because the service-under-test (SUT) doesn't concern itself with the authentication process. Also, we don't want to test with a real instance of Access Token because it typically contains a lot more information than what we need to test (e.g. that an endpoint is protected by a scope, we just want to test that endpoint with an Access Token that only has that particular scope) which the authorization test inconclusive. So, we [mock the Authorization Server](https://auth0.com/blog/xunit-to-test-csharp-code/#Mocking-External-Dependencies).

2. Database connections typically have to be re-configured using the code sample below if using Docker-in-Docker, which happens when the tests are executed in a container, so services spun up through `TestContainers` are running as containers within that test container. In this scenario, it's not possible to configure the database connection string during compile-time because the hostname and port number are unknown. So the connection string has to be obtained using `TestContainers` API e.g. `Hostname` and `GetMappedPublicPort` ([reference](https://dotnet.testcontainers.org/api/best_practices/)):

    ```csharp
    var dbContextDescriptor = services.Single(serviceDescriptor => serviceDescriptor.ServiceType == typeof(DbContextOptions<MyType>));
    services.Remove(dbContextDescriptor);
    services.AddDbContext<MyType>(options => options.UseNpgsql...);
    ```

3. `TestContainers` and `WireMock` are initialized through static classes with static variables and static constructors, so the containers are started before any of the tests run. xUnit supports [Assembly Fixtures](https://xunit.net/docs/shared-context) in v3 but it isn't released yet (in beta). Same for WireMock (though sometimes you don't need the WireMock server to be running throughout the entire test execution if only a few tests are using the WireMock server).

4. `Respawner` saves lives. Generally, opt for auto-deletion of state that runs at the end of each test (through the `Dispose` method).

5. Use a `appsettings.Test.json` file to set default values that you will probably not change across most / all tests.

### Logging

xUnit doesn't provide functionality to view logs emitted by the `TestServer`. To do so, the `TestServer` has to be configured to output logs using `ITestOutputHelper` (a dependency that's injected in the test class' `ctor` parameters, see [guide](https://xunit.net/docs/capturing-output)). There's 2 ways to do so:

1. Use [Meziantou.Extensions.Logging.Xunit](https://www.meziantou.net/how-to-get-asp-net-core-logs-in-the-output-of-xunit-tests.htm) (on a side note, because the library's code is simple, it's also possible to not use this library and simply copy the source code). It contains implementations of `ILogger`, `ILogger<T>`, `ILoggerProvider` that output the logs to `ITestOutputHelper`. Sample code:

    ```csharp
    public class CustomWebApplicationFactory(ITestOutputHelper OutputHelper) : WebApplicationFactory<Program>
    {
        protected override void ConfigureWebHost(IWebHostBuilder builder)
        {
            builder.ConfigureLogging(loggingBuilder => loggingBuilder.Services.AddSingleton<ILoggerProvider>(
                serviceProvider => new XUnitLoggerProvider(OutputHelper)));
        }
    }

    public class MyTestClass : IDisposable
    {
        private readonly CustomWebApplicationFactory Factory;

        public MyTestClass(ITestOutputHelper outputHelper)
        {
            Factory = new CustomWebApplicationFactory(outputHelper);
        }

        public void Dispose() => Factory.Dispose();
    }
    ```

    The main con of this approach is it doesn't support [Class Fixtures and Collection Fixtures](https://xunit.net/docs/shared-context) when the `CustomWebApplicatonFactory` should be shared across multiple tests instead of being re-created per test. In these scenarios, the `CustomWebApplicationFactory` is injected as a dependency in the test class' `ctor` parameters; the `TestServer` is already running before the test class' `ctor` runs.

    Previously, I tried using `Reflection` to set `ITestOutputHelper` (in `ConfigureWebHost`, call `new XUnitLoggerProvider(null)`, then set the real instance in the test class' `ctor` using `Reflection`). However, this resulted in all tests after the first to not emit logs. I am unsure why this is the case, but it could be due to ASP.NET Core applications re-using the same instance of `ILogger` that's created when the first test was executed; Even though subsequent tests set the new instance of `ITestOutputHelper` in `XUnitLoggerProvider`, no new instances of `ILogger` are created, and the cached `ILogger` is still using the first test's instance of `ITestOutputHelper`.

2. Use [MartinCostello.Logging.XUnit](https://github.com/martincostello/xunit-logging). This library supports Class Fixtures and Collection Fixtures, and so this should be the better solution. Sample code:

    ```csharp
    [CollectionDefinition("foo")]
    public class CollectionFixture : ICollectionFixture<Fixture>;

    public class Fixture : WebApplicationFactory<Program>, ITestOutputHelperAccessor
    {
        public ITestOutputHelper? OutputHelper { get; set; }

        protected override void ConfigureWebHost(IWebHostBuilder builder)
            => builder.ConfigureLogging(loggingBuilder => loggingBuilder.AddXUnit(this));
    }

    [Collection("foo")]
    public class MyTestClass : IDisposable
    {
        private readonly CustomWebApplicationFactory Factory;

        public MyTestClass(CollectionFixture fixture, ITestOutputHelper outputHelper)
        {
            fixture.OutputHelper = outputHelper;
        }

        public void Dispose() => Factory.Dispose();
    }
    ```

## Code Organization

There are 3 key entities:

1. [Collection Fixtures](https://xunit.net/docs/shared-context): Allows sharing of the same instance of `*WebApplicationFactory` across multiple tests instead of spinning up a new instance per test. All `*Fixture` must extend `FixtureBase<T>`, where generic (`T`) is used to initialize and dispose the `*WebApplicationFactory`.

2. Test classes: All test classes must extend a `*TestBase` class, which must extend `TestBase<T>` which provides common code for:
    1. Initialization and tearing down of shared functionality e.g. database, message queues, configuring logging
    2. Helper methods
    3. Generic (`T`) is used to inject dependency `FixtureBase<T>` to expose common properties like `HttpClient`.

3. `WebApplicationFactory`: All `*WebApplicationFactory` must extend `WebApplicationFactoryBase`.

How these classes relate to each other: A Test class injects a Collection Fixture dependency containing a `WebApplicationFactory`.

### Sample Code

```csharp
public abstract class FixtureBase<T> : IDisposable where T : WebApplicationFactoryBase, new()
{
    public readonly T Factory;
    public readonly HttpClient Client;

    protected FixtureBase()
    {
        Factory = new T();
        Client = Factory.CreateClient();
    }

    public void Dispose() => Factory.Dispose();
}

public abstract class TestBase<T> : IAsyncLifetime where T : WebApplicationFactoryBase, new()
{
    T Factory;
    HttpClient Client;

    protected TestBase(FixtureBase<T> fixture, ITestOutputHelper output)
    {
        Factory = fixture.Factory;
        Client = fixture.Client;
        Factory.OutputHelper = output;
    }

    public Task InitializeAsync() => Task.CompletedTask;
    async Task IAsyncLifetime.DisposeAsync()
    {
        Factory.OutputHelper = null;
        // ...
    }

    {type} GetOptions()
    {
        return Factory.Services.GetRequiredService<IOptions<{type}>>().Value;
    }

    void SetOptions(Action<{type}> action)
    {
        var options = Factory.Services.GetRequiredService<IOptions<{type}>>().Value;
        action.Invoke(options);
    }
}

public abstract class WebApplicationFactoryBase : WebApplicationFactory<Program>, ITestOutputHelperAccessor
{
    public ITestOutputHelper? OutputHelper { get; set; }

    protected override IHostBuilder? CreateHostBuilder()
    {
        // Has to be called here to set `context.HostingEnvironment.EnvironmentName`;
        // it is too late to call it at ConfigureWebHost(IWebHostBuilder) 
        Environment.SetEnvironmentVariable("ASPNETCORE_ENVIRONMENT", "Test");
        return base.CreateHostBuilder();
    }

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        // The following segment contains the bare minimum configuration changes required to start up the server;
        // so, the test instance will be exactly like the actual instance of the server.
        builder.ConfigureAppConfiguration((_, config) =>
        {
            config.AddJsonFile(Constants.TestAppSettingsFilePath(Assembly.GetExecutingAssembly().Location), false);
        });
    
        builder.ConfigureServices(services =>
        {
            var configuration = services.BuildServiceProvider().GetRequiredService<IConfiguration>();
            // Set configuration values, specifically for those that cannot be configured in the appsettings file e.g. TestContainers hostname and ports

            var dbContextDescriptor = services.Single(serviceDescriptor => serviceDescriptor.ServiceType == typeof(DbContextOptions<{type}>));
            services.Remove(dbContextDescriptor);
            services.AddDbContext<{type}>(options => options.UseNpgsql...);
        });

        // The following segment contains configuration changes for ease of testing
        builder.ConfigureLogging(loggerBuilder => loggerBuilder.AddXUnit(this));

        builder.ConfigureServices(services =>
        {
            // decouples the service from the actual instance of the Auth Server
            services.PostConfigure<JwtBearerOptions>(JwtBearerDefaults.AuthenticationScheme, options
            {
                options.TokenValidationParameters = new TokenValidationParameters
                {
                    IssuerSigningKey = JwtClaimsBuilder.SecurityKey,
                    ValidIssuer = JwtClaimsBuidler.Issuer,
                    ValidAudience = JwtClaimsBuilder.Audience
                };
            });
        });
    }
}

// https://auth0.com/blog/xunit-to-test-csharp-code/#Mocking-External-Dependencies
public class JwtClaimsBuilder
{
    string Issuer = Guid.NewGuid().ToString();
    string Audience = Guid.NewGuid().ToString();
    byte[] Key = Enumerable.Repeat((byte)0x0, 32).ToArray();
    SecurityKey SecurityKey = new SymmetricSecurityKey(Key) { KeyId = Guid.NewGuid().ToString() };
    SigningCredentials SigningCredentials = new(SecurityKey, SecurityAlgorithms.HmacSha256);
    JwtSecurityTokenHandler TokenHandler = new();
    List<Claim> Claims = [];

    string GenerateJwt()
    {
        return TokenHandler.WriteToken(new JwtSecurityToken(Issuer, Audience, Claims, null, DateTime.UtcNow.AddMinutes(10), SigningCredentials));
    }
}

// test code
public class TypicalFixture : FixtureBase<TypicalWebApplicationFactory>;

[CollectionDefinition("TypicalTest")]
public class TypicalCollection : ICollectionFixture<TypicalFixture>;

[Collection("TypicalTest")]
public abstract class TypicalTestBase(TypicalFixture fixture, ITestOutputHelper output)
    : TestBase<TypicalWebApplicationFactory>(fixture, output);

public class TypicalWebApplicationFactory : WebApplicationFactoryBase;

public class MyTest(TypicalFixture fixture, ITestOutputHelper output) : TypicalTestBase(fixture, output)
{
    // ...
}
```
