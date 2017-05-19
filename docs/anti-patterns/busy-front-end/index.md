---
title: Busy Front End antipattern
description: 
author: dragon119
---

# Busy Front End antipattern

Performing asynchronous work in a large number of background threads can starve other concurrent foreground tasks of resources, decreasing response times to unacceptable levels.

## Context

Resource-intensive tasks can increase response times for user requests and cause high latency. One way to improve response times is to offload a resource-intensive task to a separate thread. This strategy lets the application stay responsive while processing happens in the background. However, tasks that run on a background thread still consume resources. If there are too many, they can starve the threads that are handling requests.

> [!NOTE]
> The term *resource* can encompass many things, such as CPU utilization, memory occupancy, and network or disk I/O.

This problem typically occurs when an application is developed as single monolithic piece of code, with the entire business processing combined into a single tier shared with the presentation layer.

Here’s an example that demonstrates the problem. The example uses ASP.NET Web API. You can find the complete sample [here][code-sample].

- The `Post` method in the `WorkInFrontEnd` controller implements an HTTP POST operation. This operation simulates a long-running, CPU-intensive task. The work is performed on a separate thread, in an attempt to enable the POST operation to complete quickly and ensure that the caller remains responsive.

- The `Get` method in the `UserProfile` controller implements an HTTP GET operation. This method is much less CPU intensive.


```csharp
public class WorkInFrontEndController : ApiController
{
    [HttpPost]
    [Route("api/workinfrontend")]
    public void Post()
    {
        new Thread(() =>
        {
            //Simulate processing
            Thread.SpinWait(Int32.MaxValue / 100);
        }).Start();

        return Request.CreateResponse(HttpStatusCode.Accepted);
    }
}

public class UserProfileController : ApiController
{
    [HttpGet]
    [Route("api/userprofile/{id}")]
    public UserProfile Get(int id)
    {
        //Simulate processing
        return new UserProfile() { FirstName = "Alton", LastName = "Hudgens" };
    }
}
```

The primary concern is the resource requirements of the Post method. Although it puts the work onto a background thread, the work can still consume considerable CPU resources. These resources are shared with other operations being performed by other concurrent users. If a moderate number of users send this request at the same time, overall performance is likely to suffer, slowing down all operations. Users might experience significant latency in the `Get` method, for example.

## How to correct the problem

Move processes that consume significant resources to a separate back end. 

With this approach, the front end puts resource-intensive tasks onto a message queue. The back end picks up the tasks for asynchronous processing. The queue also acts as a natural load leveler, buffering requests for the back end. If the queue length becomes too long, you can configure autoscaling to scale out the back end.

Here is a revised version of the previous code. In this version, the `Post` method puts a message on a Service Bus queue. 

```csharp
public class WorkInBackgroundController : ApiController
{
    private static readonly QueueClient QueueClient;
    private static readonly string QueueName;
    private static readonly ServiceBusQueueHandler ServiceBusQueueHandler;

    public WorkInBackgroundController()
    {
        var serviceBusConnectionString = ...;
        QueueName = ...;
        ServiceBusQueueHandler = new ServiceBusQueueHandler(serviceBusConnectionString);
        QueueClient = ServiceBusQueueHandler.GetQueueClientAsync(QueueName).Result;
    }

    [HttpPost]
    [Route("api/workinbackground")]
    public async Task<long> Post()
    {
        return await ServiceBusQueuehandler.AddWorkLoadToQueueAsync(QueueClient, QueueName, 0);
    }
}
```

The back end pulls messages from the Service Bus queue and does the processing.

```csharp
public async Task RunAsync(CancellationToken cancellationToken)
{
    this._queueClient.OnMessageAsync(
        // This lambda is invoked for each message received.
        async (receivedMessage) =>
        {
            try
            {
                // Simulate processing of message
                Thread.SpinWait(Int32.Maxvalue / 1000);

                await receivedMessage.CompleteAsync();
            }
            catch
            {
                receivedMessage.Abandon();
            }
        });
}
```

## Considerations

- This approach adds some additional complexity to the application. You must handle queuing and dequeuing safely to avoid losing requests in the event of a failure.
- The application takes a dependency on an additional service for the message queue.
- The processing environment must be sufficiently scalable to handle the expected workload and meet the required throughput targets.

## How to detect the problem

Symptoms of a busy front end include high latency when resource-intensive tasks are being performed. End users are likely to report extended response times and possible failures caused by services timing out. These failures could also return HTTP 500 (Internal Server) errors or HTTP 503 (Service Unavailable) errors. Examine the event logs for the web server, which are likely to contain more detailed information about the causes and circumstances of the errors.

You can perform the following steps to help identify this problem:

1. Perform process monitoring of the production system, to identify points when response times slow down.
2. Examine the telemetry data captured at these points to determine the mix of operations being performed and the resources being utilized. Find any correlations between poor response times and the volumes and combinations of operations that were happening at those times.
3. Perform load testing of each possible operation to identify which operations are consuming resources and starving other operations. 
4. Review the source code for those operations to determine why they might cause excessive resource consumption.

If you already have insight into the problem, you may be able to skip some of these steps. However, avoid making unfounded or biased assumptions. A thorough analysis can sometimes find unexpected causes of performance problems. 

## Example diagnosis 

The following sections apply these steps to the sample application described earlier.

### Identify points of slowdown

Instrument each method to track the duration and resources consumed by each request. Then monitor the application in production. This can provide an overall view of how requests compete with each other. During periods of stress, slow-running resource-hungry requests will likely impact other operations, and this behavior can be observed by monitoring the system and noting the drop off in performance.

The following image shows a monitoring dashboard. (We used [AppDyanamics] for our tests.) Initially, the system has light load. Then users start requesting the `UserProfile` GET method. The performance is reasonably good until other users start issuing requests to the `WorkInFrontEnd` POST method. At that point, response times suddenly increase dramatically (first arrow). The response time only improves after the volume of requests to the `WorkInFrontEnd` controller diminishes (second arrow).

![AppDynamics Business Transactions pane showing the effects of the response times of all requests when the WorkInFrontEnd controller is used][AppDynamics-Transactions-Front-End-Requests]

### Examine telemetry data and find correlations

The next image shows some of the metrics gathered to monitor resource utilization during the same interval. At first, few users are accessing the system. As more users connect, CPU utilization becomes very high (100%). Also notice that the network I/O rate initialy goes up as the CPU utilization rises. But once CPU usage peaks, network I/O actually goes down. That's because the system can only handle a relatively small number of requests once the CPU is at capacity. As users disconnect, the CPU load tails off.

![AppDynamics metrics showing the CPU and network utilization][AppDynamics-Metrics-Front-End-Requests]

At this point, it appears the `Post` method in the `WorkInFrontEnd` controller is a prime candidate for closer examination. Further work in a controlled environment is needed to confirm the hypothesis.

### Perform load testing to identify bad actors

The next step is to perform tests in a controlled environment. For example, run a series of load tests that include and then omit each request in turn to see the effects.

The graph below shows the results of a load test performed against an identical deployment of the cloud service used in the previous tests. The test used a constant load of 500 users performing the `Get` operation in the `UserProfile` controller, along with a step load of users performing the `Post` operation in the `WorkInFrontEnd` controller. 

![Initial load test results for the WorkInFrontEnd controller][Initial-Load-Test-Results-Front-End]

Initially, the step load is 0, so the only active users are performing the `UserProfile` requests. The system is able to respond to approximately 500 requests per second. After 60 seconds, a load of 100 additional users starts sending POST requests to the `WorkInFrontEnd` controller. Almost immediately, the workload sent to the `UserProfile` controller drops to about 150 requests per second. This is due to the way the load-test runner functions. It waits for a response before sending the next request, so the longer it takes to receive a response, the lower the request rate.

As more users send POST requests to the `WorkInFrontEnd` controller, the response rate of the `UserProfile` controller continues to drop. But note that the volume of requests handled by the `WorkInFrontEnd`controller remains relatively constant. The saturation of the system becomes apparent as the overall rate of both requests tends towards a steady but low limit.


### Review the source code

The final step is to look at the source code. When the 

In the case of the `Post` method in the `WorkInFrontEnd` controller, the
development team was aware that this request could take a considerable amount of time
which is why the processing is performed on a different thread running asynchronously.
In this way a user issuing the request does not have to wait for processing to
complete before being able to continue with the next task.

```csharp
public void Post()
{
    new Thread(() =>
    {
        //Simulate processing
        Thread.SpinWait(Int32.MaxValue / 100);
    }).Start();

    return Request.CreateResponse(HttpStatusCode.Accepted);
}
```

However, although this approach theoretically improves response time for the user, it
introduces a small overhead associated with creating and managing a new thread.
Additionally, the work performed by this method still consumes CPU, memory, and other
resources. Enabling this process to run asynchronously might actually be damaging to
performance as users can possibly trigger a large number of these operations
simultaneously, in an uncontrolled manner. In turn, this has an effect on any other
operations that the server is attempting to perform. Furthermore, there is a finite
limit to the number of threads that a server can run, determined by a number of
factors such as available computing resource capacity and the number of CPU cores.
When the system reaches its limit, applications are likely to receive an exception
when they attempt to start a new thread.


### Implement the solution and verify the result

Running the [sample solution][fullDemonstrationOfSolution] in a production environment
and using AppDynamics to monitor performance generated the following results. The load
was similar to that shown earlier, but the response times of requests to the
`UserProfile` controller is now much faster and the volume of requests overall was
greatly increased over the same duration (23565 against 2759 earlier). A much bigger
volume of requests was made to the `WorkInBackground` controller compared to the
earlier tests due to the increased efficiency of these requests. However, you should
not compare the throughput and response time for these requests to those shown earlier
as the work being performed is very different (queuing a request rather than
performing a time consuming calculation).

![AppDynamics Business Transactions pane showing the effects of the response times of all requests when the WorkInBackground controller is used][AppDynamics-Transactions-Background-Requests]

The CPU and network utilization also illustrate the improved performance. The CPU
utilization never reached 100% and the volume of network requests handled was far
greater than earlier and did not tail off until the workload dropped.

![AppDynamics metrics showing the CPU and network utilization for the WorkInBackground controller][AppDynamics-Metrics-Background-Requests]

Repeating the controlled load test over 5 minutes for users submitting a mixture of
all requests against the `UserProfile` and `WorkInBackground` controllers gives the
following results.

![Load-test results for the BackgroundImageProcessing controller][Load-Test-Results-Background]

This graph confirms the improvement in performance of the system as a result of
offloading the intensive processing to the worker role. The overall volume of requests
serviced is greatly improved compared to the earlier tests.

Relocating the resource-hungry workload to a separate set of processes should improve
responsiveness for most requests, but the resource-hungry tasks may take
longer (this duration is not illustrated in the two graphs above, and requires
instrumenting and monitoring the worker role.) <<RBC: Is there a simpler way to state the previous sentence? Does my rewrite work? It felt like there were a lot of process/processing so I changed some.>> If there are insufficient worker role
instances available to perform the resource-hungry workload, jobs might be queued or
otherwise held pending for an indeterminate period. However, it might be possible to
expedite critical jobs that must be performed quickly by using a priority queuing
mechanism.

## Related guidance

- Background Jobs
- Queue based load leveling
- Autoscaling
- Web Queue Worker architecture style


[AppDyanamics]: https://www.appdynamics.com/
[code-sample]: https://github.com/mspnp/performance-optimization/tree/master/BusyFrontEnd
[fullDemonstrationOfSolution]: https://github.com/mspnp/performance-optimization/tree/master/BusyFrontEnd
[WebJobs]: http://www.hanselman.com/blog/IntroducingWindowsAzureWebJobs.aspx
[ComputePartitioning]: https://msdn.microsoft.com/library/dn589773.aspx
[ServiceBusQueues]: https://msdn.microsoft.com/library/azure/hh367516.aspx
[AppDynamics-Transactions-Front-End-Requests]: ./_images/AppDynamicsPerformanceStats.jpg
[AppDynamics-Metrics-Front-End-Requests]: ./_images/AppDynamicsFrontEndMetrics.jpg
[Initial-Load-Test-Results-Front-End]: ./_images/InitialLoadTestResultsFrontEnd.jpg
[AppDynamics-Transactions-Background-Requests]: ./_images/AppDynamicsBackgroundPerformanceStats.jpg
[AppDynamics-Metrics-Background-Requests]: ./_images/AppDynamicsBackgroundMetrics.jpg
[Load-Test-Results-Background]: ./_images/LoadTestResultsBackground.jpg


