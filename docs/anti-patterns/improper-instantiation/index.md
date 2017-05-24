---
title: Improper Instantiation antipattern
description: 
author: dragon119
---

# Improper Instantiation antipattern



## Problem description

Many .NET Framework libraries provide abstractions of external resources. Internally, these classes typically manage their own connections to the resource, acting as brokers that clients can use to access the resource. Here are some examples of this type of broker class that are relevant to Azure applications:

- `System.Net.Http.HttpClient`. Communicate with a web service using HTTP.
- `Microsoft.ServiceBus.Messaging.QueueClient`. Posting and receiving messages to a Service Bus queue. 
- `Microsoft.Azure.Documents.Client.DocumentClient`. Connect to a Cosmos DB instance
- `StackExchange.Redis.ConnectionMultiplexer`. Connect to Azure Redis Cache.

These broker classes can be expensive to create. They are intended to be instantiated once and reused throughout the lifetime of an application. However, it's a common misunderstanding that these classes should acquired only as necessary and released quickly.

The following ASP.NET example creates an instance of `HttpClient` to communicate with a remote service. You can find the complete sample [here][sample-app].

```csharp
public class NewHttpClientInstancePerRequestController : ApiController
{
    // This method creates a new instance of HttpClient and disposes it for every call to GetProductAsync.
    public async Task<Product> GetProductAsync(string id)
    {
        using (var httpClient = new HttpClient())
        {
            var hostName = HttpContext.Current.Request.Url.Host;
            var result = await httpClient.GetStringAsync(string.Format("http://{0}:8080/api/...", hostName));
            return new Product { Name = result };
        }
    }
}
```

In a web application, this technique is not scalable. A new `HttpClient` object is created for each user request. Under heavy load, the web server may exhaust the number of available sockets, resulting in `SocketException` errors.

This problem is not restricted to the `HttpClient` class. Other classes that wrap resources or are expensive to create might cause similar issues. The following example creates an instances of the `ExpensiveToCreateService` class. Here the issue is not necessarily socket exhaustion, but simply how long it takes to create each instance. Continually creating and destroying instances of this class might adversely affect the scalability of the system.

```csharp
public class NewServiceInstancePerRequestController : ApiController
{
    public async Task<Product> GetProductAsync(string id)
    {
        var expensiveToCreateService = new ExpensiveToCreateService();
        return await expensiveToCreateService.GetProductByIdAsync(id);
    }
}

public class ExpensiveToCreateService
{
    public ExpensiveToCreateService()
    {
        // Simulate delay due to setup and configuration of ExpensiveToCreateService
        Thread.SpinWait(Int32.MaxValue / 100);
    }
    ...
}
```

## How to fix the problem

If the class that wraps the external resource is shareable and thread-safe, create a shared singleton instance or a pool of reusable instances of the class.

The following following example uses a static `HttpClient` instance, thus sharing the connection across all requests.

```csharp
public class SingleHttpClientInstanceController : ApiController
{
    private static readonly HttpClient HttpClient;

    static SingleHttpClientInstanceController()
    {
        HttpClient = new HttpClient();
    }

    // This method uses the shared instance of HttpClient for every call to GetProductAsync.
    public async Task<Product> GetProductAsync(string id)
    {
        var hostName = HttpContext.Current.Request.Url.Host;
        var result = await HttpClient.GetStringAsync(string.Format("http://{0}:8080/api/...", hostName));
        return new Product { Name = result };
    }
}
```

## Considerations

- The key element of this antipattern is repeatedly creating and destroying instances of a *shareable* object. If a class is not shareable (not thread-safe), then this antipattern does not apply.

- The type of shared resource might dictate whether you should use a singleton or create a pool. The `HttpClient` class
is designed to be shared rather than pooled. Other objects might support pooling, enabling the system to spread the workload across multiple instances.

- Objects that you share across multiple requests *must* be thread-safe. The `HttpClient` class is designed to be used in this manner, but other classes might not support concurrent requests, so check the available documentation.

- Some resource types are scarce and should not be held onto. Database connections are an example. Holding an open database connection that is not required may prevent other concurrent users from gaining access to the database.

- In the .NET Framework, many objects that establish connections to external resources are created by using static factory methods of other classes that manage these connections. The objects intended to be saved and reused, rather than disposed and recreated. For example, in Azure Service Bus, the `QueueClient` object is created through a `MessagingFactory` object. Internally, the `MessagingFactory` managems connections. (See [Best Practices for performance improvements using Service Bus Messaging][service-bus-messaging]).


## How to detect the problem

Symptoms of this problem include a drop in throughput, possibly with an increase in exceptions indicating exhaustion of related resources such as sockets, database connections, file handles, and so on. 

You can perform the following steps to help identify this problem:

1. Identify points at which response times slow down or the system fails due to lack
of resources, by performing process monitoring of the production system.

2. Examine the telemetry data captured at these points to determine which operations
might be creating and destroying resource-consuming objects as the system slows down
or fails.

3. Perform load testing of each of the operations identified by step 2. Use a
controlled test environment rather than the production system.

4. Review the source code for the possible problematic <<RBC: Just saying possible seems too vague in this context. Does this work, or is there a better way to phrase this from a technical perspective?>> operations and examine the logic behind the
the lifecycle of the broker objects being created and destroyed.

The following sections apply these steps to the sample application described earlier.

----------

**Note:** If you already have an insight into where problems might lie, you may be
able to skip some of these steps. However, you should avoid making unfounded or biased
assumptions. Performing a thorough analysis can sometimes lead to the identification
of unexpected causes of performance problems. The following sections are formulated to
help you to examine applications and services systematically.

----------

### Identifying points of slow down or failure

Instrumenting each operation in the production system to track the duration of each
request and then monitoring the live system can help to provide an overall view of how
the requests perform. You should monitor the system and track which operations are
long-running or cause exceptions.

The following image shows the Overview pane of the New Relic <<RBC: If you decide to do something based on my comments in other docs, this should match.>> monitor dashboard,
highlighting operations that have a poor response time and the increased error rate
that occurs while these operations are running. This telemetry was gathered while the
system was undergoing load testing, but similar observations are likely to occur in
the live system during periods of high usage. In this case, operations that
invoke the `GetProductAsync` method in the `NewHttpClientInstancePerRequest`
controller are worth investigating further.

![The New Relic monitor dashboard showing the sample application creating a new instance of an HttpClient object for each request][dashboard-new-HTTPClient-instance]

You should also look for operations that trigger increased memory use and garbage
collection, as well as raised levels of network, disk, or database activity as
connections are made, files are opened, or database connections established.

### Examining telemetry data and finding correlations

You should examine stack trace information for operations that have been identified as
slow-running or that generate exceptions when the system is under load. This
information can help to identify how these operations are utilizing resources, and
exception information can be used to determine whether errors are caused by the system
exhausting shared resources. The image below shows data captured using thread
profiling for the period corresponding to that shown in the previous image. Note that
the system spends a significant time opening socket connections and even more time
closing them and handling socket exceptions.

![The New Relic thread profiler showing the sample application creating a new instance of an HttpClient object for each request][thread-profiler-new-HTTPClient-instance]

### Performing load testing

You can use load testing based on workloads that emulate the typical sequence of
operations that users might perform to help identify which parts of a system suffer
from resource exhaustion under varying loads. You should perform these tests in a
controlled environment rather than the production system. The following graph shows
the throughput of requests directed at the `NewHttpClientInstancePerRequest`
controller in the sample application as the user load is increased up to 100  
concurrent users.

![Throughput of the sample application creating a new instance of an HttpClient object for each request][throughput-new-HTTPClient-instance]

The volume of requests handled per second increases at the 10-user point due to the
increased workload up to approximately 30 users. At this point, the volume of
successful requests reaches a limit and the system starts to generate exceptions. The
volume of these exceptions gradually increases with the user load. These failures are
reported by the load test as HTTP 500 (Internal Server) errors. Reviewing the
telemetry (shown earlier) for this test case reveals that these errors are caused by
the system running out of socket resources as more and more `HttpClient` objects are
created.

The second graph below shows the results of a similar test performed using the
`NewServiceInstancePerRequest` controller. This controller does not use `HttpClient`
objects, but instead utilizes a custom object (`ExpensiveToCreateService`) to fetch
data.

![Throughput of the sample application creating a new instance of the ExpensiveToCreateService for each request][throughput-new-ExpensiveToCreateService-instance]

This time, although the controller does not generate any exceptions, throughput still
reaches a plateau while the average response time increases by a factor of 20 with
user load. *Note that the scale for the response time and throughput are logarithmic,
so the rate at which the response time grows is actually more dramatic than appears at
first glance.* Examining the telemetry for this code reveals that the main causes of
this limitation are the time and resources spent creating new instances of the
`ExpensiveToCreateService` for each request.

### Reviewing the code

Once you have managed to identify which parts of an application are causing the system
to slow down or generate exceptions due to resource exhaustion, perform a review of
the code or use profiling to find out how shareable objects are being instantiated,
used, and destroyed. Where appropriate, refactor the code to cache and reuse objects,
as described in the following section.

## Consequences of the solution

The system should be more scalable, offer a higher throughput (the system is wasting less time
acquiring and releasing resources and is therefore able to spend more time doing useful work),
and report fewer errors as the workload increases. The following graph shows the load test
results for the sample application, using the same workload as before, but invoking the
`GetProductAsync` method in the `SingleHttpClientInstance` controller.

![Throughput of the sample application reusing the same instance of an HttpClient object for each request][throughput-single-HTTPClient-instance]

No errors were reported, and the system was able to handle an increasing load of up to
 <<RBC: It can't be up to and over. I'm guessing up to is correct.>> 500 requests per second. The volume of requests capable of being handled closely mirroring
the user load. The average response time was close to half that of the previous test. The result
is a system that is much more scalable than before.

The next graph shows the results of the equivalent load test for the `SingleServiceInstance`
controller. *Note that as before, the scale for the response time and throughput for this graph
are logarithmic.*

![Throughput of the sample application reusing the same instance of an HttpClient object for each request][throughput-single-ExpensiveToCreateService-instance]

The volume of requests handled increases in line with the user load while the average response
time remains low. This is similar to the profile of the code that creates a single `HttpClient`
instance.

For comparison purposes with the earlier test, the following image shows the stack trace
telemetry for the `SingleHttpClientInstance` controller. This time the system spends most of its
time performing real work rather than opening and closing sockets.

![The New Relic thread profiler showing the sample application creating single instance of an HttpClient object for all requests][thread-profiler-single-HTTPClient-instance]


## Related resources

- [Best Practices for Performance Improvements Using Service Bus Brokered Messaging][best-practices-service-bus].

- [Performance Tips for Azure DocumentDB - Part 1][performance-tips-documentdb]

- [Redis ConnectionMultiplexer - Basic Usage][redis-multiplexer-usage]

[sample-app]: https://github.com/mspnp/performance-optimization/tree/master/ImproperInstantiation


[service-bus-messaging]: /service-bus-messaging/service-bus-performance-improvements
[performance-tips-documentdb]: http://blogs.msdn.com/b/documentdb/archive/2015/01/15/performance-tips-for-azure-documentdb-part-1.aspx
[redis-multiplexer-usage]: https://github.com/StackExchange/StackExchange.Redis/blob/master/Docs/Basics.md
[NewRelic]: http://newrelic.com/azure
[AppDynamics]: http://www.appdynamics.co.uk/cloud/windows-azure
[throughput-new-HTTPClient-instance]: _images/HttpClientInstancePerRequest.jpg
[dashboard-new-HTTPClient-instance]: _images/HttpClientInstancePerRequestWebTransactions.jpg
[thread-profiler-new-HTTPClient-instance]: _images/HttpClientInstancePerRequestThreadProfile.jpg
[throughput-new-ExpensiveToCreateService-instance]: _images/ServiceInstancePerRequest.jpg
[throughput-single-HTTPClient-instance]: _images/SingleHttpClientInstance.jpg
[throughput-single-ExpensiveToCreateService-instance]: _images/SingleServiceInstance.jpg
[thread-profiler-single-HTTPClient-instance]: _images/SingleHttpClientInstanceThreadProfile.jpg
