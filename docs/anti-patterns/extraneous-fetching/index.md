---
title: Extraneous Fetching anitpattern
description: 
author: dragon119
---

# Extraneous Fetching antipattern

Retrieving more data than is necessary to satisfy a business operation can result in unnecessary I/O overhead and reduce responsiveness. 

## Problem description

This antipattern can occur if the application tries to minimize I/O requests by retrieving all of the data that it *might* need. This is often a result of overcompensating for the [Chatty I/O][chatty-io] antipattern. For example, an application might fetch the details for every product in a database. But the user may need just a subset of the details (some details may not be relevant to customers), and probably doesn't need to see *all* of the products. Even if the user is browsing the entire catalog, it would make sense to paginate the results &mdash; showing 20 at a time, for example.

Another source of this problem is following poor programming or design practices. For example, the following code uses Entity Framework to fetch the complete details for every product, then filters it to return only a subset of the fields, discarding the rest. 

```csharp
public async Task<IHttpActionResult> GetAllFieldsAsync()
{
    using (var context = new AdventureWorksContext())
    {
        // Execute the query. This happens at the database.
        var products = await context.Products.ToListAsync();

        // Project fields from the query results. This happens in application memory.
        var result = products.Select(p => new ProductInfo { Id = p.ProductId, Name = p.Name });
        return Ok(result);
    }
}
```

Here is an example where the application retrieves data to perform an aggregation, which could be done by the database instead. The application is calculating total sales by getting every record for all orders sold, and then calculating the total sales value from those records.

```csharp
public async Task<IHttpActionResult> AggregateOnClientAsync()
{
    using (var context = new AdventureWorksContext())
    {
        // Fetch all order totals from the database.
        var orderAmounts = await context.SalesOrderHeaders.Select(soh => soh.TotalDue).ToListAsync();

        // Sum the order totals in memory.
        var total = orderAmounts.Sum();
        return Ok(total);
    }
}
```

The next example shows a subtle problem caused by the way Entity Framework uses LINQ to Entities. 

```csharp
var query = from p in context.Products.AsEnumerable()
            where p.SellStartDate < DateTime.Now.AddDays(-7) // AddDays cannot be mapped by LINQ to Entities
            select ...;

List<Product> products = query.ToList();
```

Here the application is trying to find products with a `SellStartDate` more than a week old. In most cases, LINQ to Entities would translate a `where` clause to a SQL statement, to be executed by the database. In this case, however, LINQ to Entities cannot map the `AddDays` method to SQL. Instead, every row from the `Product` table is returned, and the results are filtered in memory. The call to `AsEnumerable` gives a hint that there is a problem. The `AsEnumerable`  method converts the results to an `IEnumerable` interface. The `IEnumerable`  interface supports filtering, but the filtering is performed on the client side. By default, LINQ to Entities uses `IQueryable`, which passes the responsibility for filtering to the data source. 

## How to fix the problem

Only fetch the data that you need. Avoid fetching large volumes of data that may quickly become outdated or might be discarded, and only fetch the data needed for the operation being performed. 

Instead of getting all the columns from a table and then filtering them, select the columns that you need.

```csharp
public async Task<IHttpActionResult> GetRequiredFieldsAsync()
{
    using (var context = new AdventureWorksContext())
    {
        // Project fields as part of the query itself
        var result = await context.Products
            .Select(p => new ProductInfo {Id = p.ProductId, Name = p.Name})
            .ToListAsync();
        return Ok(result);
    }
}
```

Similarly, perform aggregation in the database and not in application memory.

```csharp
public async Task<IHttpActionResult> AggregateOnDatabaseAsync()
{
    using (var context = new AdventureWorksContext())
    {
        // Sum the order totals as part of the database query.
        var total = await context.SalesOrderHeaders.SumAsync(soh => soh.TotalDue);
        return Ok(total);
    }
}
```

When using Entity Framework, ensure that LINQ queries are resolved using the `IQueryable`interface and not `IEnumerable`. You may need to adjust the query to use
only functions that can be mapped to the data source. The earlier example can be refactored to remove the `AddDays` method from the query, allowing filtering to be done by the database.

```csharp
DateTime dateSince = DateTime.Now.AddDays(-7); // AddDays has been factored out. 
var query = from p in context.Products
            where p.SellStartDate < dateSince // This criterion can be passed to the database by LINQ to Entities
            select ...;

List<Product> products = query.ToList();
```

## How to detect the problem

Symptoms of extraneous fetching include high latency and low throughput. If the data is retrieved from a data store, increased contention is also probable. End users are likely to report extended response times or failures caused by services timing out. These failures could return HTTP 500 (Internal Server) errors or HTTP 503 (Service Unavailable) errors. Examine the event logs for the web server, which are likely to contain more 
detailed information about the causes and circumstances of the errors.

The symptoms of this antipattern and some of the telemetry obtained might be very similar to those of the [Monolithic Persistence antipattern][MonolithicPersistence]. 

You can perform the following steps to help identify the causes of any problems:

1. Identify slow workloads or transactions by performing load-testing, process monitoring, or other methods of capturing instrumentation data.
2. Observe any behavioral patterns exhibited by the system. Are there particular limits in terms of transactions per second or volume of users?
3. Correlate the instances of slow workloads with behavioral patterns.
4. Identify the data stores being used. For each data source, run lower level telemetry to observe the behavior of operations.
6. Identify any slow-running queries that reference these data sources.
7. Perform a resource-specific analysis of the slow-running queries and ascertain how the data is used and consumed.

Look for any of these symptoms:

- Frequent, large I/O requests made to the same resource or data store.
- Contention in a shared resource or data store.
- An operation that frequently receives large volumes of data over the network.
- Applications and services spending significant time waiting for I/O to complete.

If you already have insight into the problem, you may be able to skip some of these steps. However, avoid making unfounded or biased assumptions. A thorough analysis can sometimes find unexpected causes of performance problems. 

## Example diagnosis    

The following sections apply these steps to the previous examples.

### Identify slow workloads

This graph shows performance results from a load test that simulated up to 400 concurrent users running the `GetAllFieldsAsync` method shown earlier. Throughput diminishes slowly as the load increases. Average response time goes up as the workload increases. 

![Load test results for the GetAllFieldsAsync method][Load-Test-Results-Client-Side1]

A load test for the `AggregateOnClientAsync` operation shows a similar pattern. The volume of requests is reasonably stable. The average response time increases with the workload, allthough more slowly than the previous graph.

![Load test results for the AggregateOnClientAsync method][Load-Test-Results-Client-Side2]

### Correlate slow workloads with behavioral patterns

Any correlation between regular periods of high usage and slowing performance can indicate areas of concern. Closely examine the performance profile of functionality that is suspected to be slow running, to determine whether it matches the load testing performed earlier.

Load test the same functionality using step-based user loads, to find the point where performance drops significantly or fails completely. If that point falls within the bounds of your expected real-world usage, examine how the functionality is implemented.

A slow operation is not necessarily a problem, if it is not being performed when the system is under stress, is not time critical, and does not negatively affect the performance of other important operations. For example, generating monthly operational statistics might be a long-running operation, but it can probably be performed as a batch process and run as a low priority job. On the other hand, customers querying the product catalog is a critical business operation. Focus on the telemetry generated by these critical operations to see how the performance varies during periods of high usage.

### Identify data sources in slow workloads

If you suspect that a service is performing poorly because of the way it retrieves data, investigate how the application interacts with the repositories it uses. Monitor the live system data to see which sources are accessed during periods of poor performance. 

For each data source, instrument the system to capture the following:

- The frequency that each data store is accessed.
- The volume of data entering and exiting the data store.
- The timing of these operations, especially the latency of requests.
- The nature and rate of any errors that occur while accessing each data store under typical load.

Compare this information against the volume of data being returned by the application to the client. Track the ratio of the volume of data returned by the data store against the volume of data returned to the client. Any large disparity between should be investigated, to determine whether the application fetching extraneous data and performing processing that the data store could handle.

![Observing end-to-end behavior of operations][End-to-End]

You may be able to capture this data by  observing the live system and tracing the lifecycle of each user request, or you can model a series of synthetic workloads and run them against a test system.

The following graphs show the telemetry captured using [New Relic APM][new-relic] during a load test of the `GetAllFieldsAsync` method. Note the difference between the volumes of data received from the database and the corresponding HTTP responses.

![Telemetry for the `GetAllFieldsAsync` method][TelemetryAllFields]

For each request, the database returned 80503 bytes, but the response to the client only contained 19855 bytes, about 25% of the size of the database response. The size of the data returned to the client can vary depending on the format. For this load test, the client requested JSON data. Separate testing using XML (not shown) had a response size of 35655 bytes, or 44% of the size of the database response.

The load test for the `AggregateOnClientAsync` method shows more extreme results. In this case, each test performed a query that
retrieved over 280Kb of data from the database, but the JSON response was a mere 14 bytes. The wide disparity is because the method calculates an aggregated result from a large volume of data.

![Telemetry for the `AggregateOnClientAsync` method][TelemetryAggregateOnClient]

### Identify slow queries

Tracing execution and analyzing the application source code and data access logic
might reveal that a number of different queries are performed as the application runs.
You should concentrate on those that consume the most resources and take the most time
to execute. You can add instrumentation to determine the start and completion times
for many database operations enabling you to work out the duration. However, many data
stores also provide in-depth information on the way in which queries are performed and
optimized. For example, the Query Performance pane in the Azure SQL Database
management portal enables you to select a query and drill into the detailed runtime
performance information for that query. The figure below shows the query generated by the `GetAllFieldsAsync` operation.

![The Query Details pane in the Windows Azure SQL Database management portal][QueryDetails]

The statistics summarize the resources used by this query.

----------

**Note:** The statistics shown in this image were not generated by running the
load tests, but were obtained by monitoring the system in production. However, the
statistics are still valid as they give an indication of how the query uses resources
and this is not dependent on whether the system is under test at the time.

----------

### Performing a resource-specific analysis of the slow-running queries

Examining the queries frequently performed against a data source, and the way an application uses this information, can provide insight into how key operations
might be speeded up. In some cases, it may be advisable to partition resources
horizontally if different attributes of the data (columns in a relational table, or
fields in a NoSQL store) are accessed separately by different functions,this can help
to reduce contention. Often 90% of operations are run against 10% of the data held in
the various data sources, so spreading this load may improve performance.

Depending on the nature of the data store, you may be able to exploit the features
that it implements to efficiently store and retrieve information. For example, if an
application requires an aggregation over a number of items (such as a count, sum, min
or max operation), SQL databases typically provide aggregate functions that can
perform these operations without requiring that an application fetches all of the data
and implement the calculation itself. In other types of data store, it may be possible
to maintain this information separately within the store as records are added,
updated, or removed, eliminating the requirement of an application to fetch a
potentially large amount of data and perform the calculation itself.

If you observe requests that retrieve a large number of fields, examine the underlying
source code to determine whether all of these fields are actually necessary. Sometimes
these requests are the result of ill-advised `SELECT *` operations, or misplaced
`.Include` operations in LINQ queries. Similarly, requests that retrieve a large
number of entities (rows in a SQL Server database) may indicate an application
that is not filtering data correctly. Verify that all of these entities are actually
necessary, and implement database-side filtering if possible (for example, using a
`WHERE` clause in an SQL statement.) For operations that have to support unbounded
queries, the system should implement pagination and only fetch a limited number (a
*page*) of entities at a time.

----------

**Note:** If analysis shows that none of these situations apply, then extraneous
fetching is unlikely to be the cause of poor performance and you should look
elsewhere.

----------


## <a name="ConsequencesOfTheSolution"></a>Consequences of the solution

The system should spend less time waiting for I/O, network traffic should be
diminished, and contention for shared data resources should be decreased. This should
show an improvement in response time and throughput in an application.
Performing load testing against the `GetRequiredFieldsAsync` method in the [sample solution][fullDemonstrationOfSolution] shows the following results:

![Load test results for the GetRequiredFieldsAsync method][Load-Test-Results-Database-Side1]

This load test was performed on the same deployment and using the same simulated
workload of 400 concurrent users as before. The graph shows much lower latency, the
response time rises with load to approximately 1.3 seconds compared to 4 seconds in
the previous case. The throughput is also higher at 350 requests per second compared
to 100 earlier. These changes are apparent from the telemetry gathered while the test
was running.

![Telemetry for the `GetRequiredFieldsAsync` method][TelemetryRequiredFields]

The volume of data retrieved from the database now closely matches the size of the
HTTP response messages sent back to the client applications.

Load testing using the `AggregateOnDatabaseAsync` method generates the following results:

![Load test results for the AggregateOnDatabaseAsync method][Load-Test-Results-Database-Side2]

The average response time is now minimal. This is an order of magnitude improvement
in performance, caused primarily by the vast reduction in I/O from the database.

The image below shows the corresponding telemetry for the `AggregateOnDatabaseAsync`
method.

![Telemetry for the `AggregateOnDatabaseAsync` method][TelemetryAggregateInDatabaseAsync]

The amount of data retrieved from the database was vastly reduced, from over 280Kb per
transaction to 53 bytes. Consequently, the maximum sustained number of requests per
minute was raised from around 2,000 to over 25,000.

----------

**Note:** The solution to this antipattern does not imply that you should always
offload processing to the database. You should only perform this strategy where the
database is designed or optimized to do so. Databases are intended to manipulate the
data that they contain very efficiently, but in many cases they are not designed to
act as complete <<RBC: Normally you'd say fullfledged, but even that seems like the easiest to parse for ESL readers. Does this work?>> application engines. Using the processing power of a database
server inappropriately can cause contention and slow down database operations. For
more information, see the [Busy Database antipattern][BusyDatabase]

----------

## Related resources

[new-relic]: https://newrelic.com/application-monitoring


[fullDemonstrationOfProblem]: https://github.com/mspnp/performance-optimization/tree/master/ExtraneousFetching

[chatty-io]: ../chatty-io/index.md
[MonolithicPersistence]: ../monolithic-persistence/index.md
[BusyDatabase]: ../busy-database/index.md
[product-sales-table]:_images/SalesOrderHeaderTable.jpg
[Load-Test-Results-Client-Side1]:_images/LoadTestResultsClientSide1.jpg
[Load-Test-Results-Client-Side2]:_images/LoadTestResultsClientSide2.jpg
[Load-Test-Results-Database-Side1]:_images/LoadTestResultsDatabaseSide1.jpg
[Load-Test-Results-Database-Side2]:_images/LoadTestResultsDatabaseSide2.jpg
[QueryPerformanceZoomed]: _images/QueryPerformanceZoomed.jpg
[QueryDetails]: _images/QueryDetails.jpg
[TelemetryAllFields]: _images/TelemetryAllFields.jpg
[TelemetryAggregateOnClient]: _images/TelemetryAggregateOnClient.jpg
[TelemetryRequiredFields]: _images/TelemetryRequiredFields.jpg
[TelemetryAggregateInDatabaseAsync]: _images/TelemetryAggregateInDatabase.jpg
