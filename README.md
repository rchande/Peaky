[![Gitter](https://badges.gitter.im/PhillipPruett/Peaky.svg)](https://gitter.im/PhillipPruett/Peaky?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)

[![Build status](https://ci.appveyor.com/api/projects/status/5ui79hatbw9k5yas/branch/master?svg=true)](https://ci.appveyor.com/project/PhillipPruett/peaky/branch/master)

[Peaky Nuget Package](https://www.nuget.org/packages/Peaky/)

### Peaky Exposes Tests as HTTP Endpoints
Peaky discovers tests in an application and maps routes to allow them to be called over HTTP. These tests can live within the application they are testing or stood up as their own service.
### What does Peaky provide me?
One of the great benefits of Peaky is moving tests away from the antiquated paradigm where tests are ran on local machines to being ran from a sharable and accessible location. This does away with _‘works on my machine’_ test issues and allows you to easily share, discover, and execute tests.

Peaky also provides users with a CDN hosted UI out of the box with hooks to provide your own UI experience if you desire.

Peaky aims to remove the barriers between production and pre-production environment tests. With Peaky you register your target environments and their dependencies and let Peaky inject the correct dependency for each test.  E.g. a tests that verifies a homepages availability needs to only be written once and depending on the request to Peaky, it can be ran against any target. 
### What is considered a Peaky test?
Basically any public method on concrete types derived from IPeakyTest. There are a few exceptions to this. Peaky will not expose the following as tests:
* Public properties
* Constructors
* Methods with **non-defaulted** parameters
* Methods with a [Special Name](https://msdn.microsoft.com/en-us/library/system.reflection.methodbase.isspecialname(v=vs.110).aspx)

### Sensors
Peaky will also discover [Its.Log Sensors](https://github.com/jonsequitur/Its.Log) in all loaded assemblies and expose them via HTTP endpoints.

### Examples
Check out the [Peaky WebApplication Sample](https://github.com/PhillipPruett/Peaky/tree/master/Sample/Peaky.SampleWebApplication) to see an example of actual tests being defined and discovered.

#### Marking a test class as a Peaky test class
Any class that implements IPeakyTest (or its children IApplyToApplication, IApplyToEnvironment, IApplyToTarget, and IHaveTags) will have its qualifying methods discovered and exposed as tests.

#### How Peaky Tests Are Written
write a peaky test much in the same way that you would write any mstest or nunit test. a test method that throws an exception will return a 500 Internal Server Error response code to the caller, signaling failure, and tests that do not throw any exceptions will return 200 OK, signaling success. 
```csharp
public string bing_homepage_returned_in_under_5ms()
        {
            var stopwatch = new Stopwatch();
            stopwatch.Start();
            httpClient.GetAsync("/").Wait();
            stopwatch.Stop();

            stopwatch.ElapsedMilliseconds.Should().BeLessThan(5);
            return $"{stopwatch.ElapsedMilliseconds} milliseconds";
        }
```
#### Routing Tests at Application Startup
Tests will by default be located at  http://yourPeakyApplicationsBaseUri**/tests**
```csharp
config.MapTestRoutes(targets => targets.Add("prod","bing", new Uri("https://bing.com")));
```
#### Structure of a Peaky Uri
a peaky uri ( http://yourPeakyApplicationsBaseUri/tests/{environment}/{application}/{testname} ) 3 major parts to it:
* Application: what is the name of the service under test
* Environment: what environment of that application is under test e.g production or internal, deployment A or deployment B.
* Test Name: the name of the method discovered as a test

These elements combine to form unique test uris. 

#### Test Discovery
All tests, their tags, and their parameters are discoverable with a query. the following are examples of queries to discover tests:

**HTTP GET /tests** will return all tests in the application
**HTTP GET /tests/{environment}** will return all tests within the requested environment
**HTTP GET /tests/{environment}/{application}** will return all tests for the application within that environment

#### Test Tags
A test class can implement IHaveTags which will allow for categorization of tests and allow users to filter based upon them:

**HTTP GET /tests/{environment}/{application}/?{tag}=true** will only return tests within the test classes with that tag
**HTTP GET /tests/{environment}/{application}/?{tag}=false** will return tests except those within the test classes with that tag

Numerous tags can be filtered on with one request.

#### Dependency Injection
by default, peaky will allow your test classes to take a dependency on an HTTP client only. this is constructed using the details provided at app startup:
```csharp
config.MapTestRoutes(targets => targets.Add("prod","bing", new Uri("https://bing.com")));
```
this would allow any test classes that apply to both 'prod' and 'bing' to depend on an httpclient.

If you have other dependencies that can be registered as follows:
```csharp
config.MapTestRoutes(targets =>
                     targets.Add("prod",
                                 "bing",
                                 new Uri("https://bing.com"),
                                 registry => registry.Register(new TableStorageClient())
                                                     .Register(new AuthenticatedHttpClient())));
```
