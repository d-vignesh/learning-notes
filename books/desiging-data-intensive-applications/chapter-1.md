## Chapter-1 : Reliable, Scalable and Maintainable Applications

A app is data-intensive, if dealing with data is its primary challenge. It may be the quantity of data, the complexity of data or the growing speed of the data. These applications need to,
- store data (database)
- speed up reads (cache)
- search by keyword or filter (search indexes)
- send a message to another process, to be handled asynchronously (stream processing)
- periodically chrunch data (batch processing)

Three aspects to consider when building such applications is reliability, scalability and maintainability.

- *Reliability*: Reliability means the systems ability to work correctly, even when things go wrong. Things going wrong can be from hardware failures(which we could handle with redundancy), software failures(which could be handled by proper testing and monitoring).

- *Scalability*: Scalability means the system's ability to cope with increased load. In order to understand the issues with might arise from scalability, first we need to identify the key load parameters in the system. For example, it might be request per second for web apps, ratio of read to writes for dbs, no. of simultaneous users incase of chat apps, hit rate on a cache, etc. 
(eg: faning-out user posted tweets to all theirs followers and constructing the home timeline for each of the follower)

- *Performance*: Once the load parameters are identified, we then need to investigate what happens to the system components when these parameters increases and how much resources of each component we will need to increase these load parameters to the required levels. The metrics to evaluate performance of a system will vary based on the nature of the system,
   - for batch processing systems, we will care about the throughput, which determines the total time the system will take.
   - for online systems, its the response time. Response time cannot be treated as a single number as we will get different response time each time we process the request. Hence it is better to use percentile mechanism when measuring the response time. In this we sort the different response timesfrom faster to slower, the median is the halfway point and is called the 50th percentile. We can consider the number of instances after say 95th, 99th, 99.9th percentiles to get a good picture of the system's performance.

Percentiles are often used in Service Layer Objectives and Service Layer Agreements(sla's) which are contracts that define the expected performance and availability of the service. For example, a SLA agreement might state that the service is considereed to be up if it has a median response time of less than 200ms and a 99th percentile under 1s(which means 99% of the requests out of the measured requests should be served within 1s) and the service should be available at-least 99.9% of time(which means the service can have a maximum downtime of only approximately 8 hours per year).

Once we understand the impact of load on the required performance, we need to find ways of coping up with the load. The common options are to scale-up/vertical-scaling(where we upgrate our existing components to more powerful machine) and scale-out/horizontal-scaling(where we add more instances and distribute the load across). Distributing stateless services across multiple machines is fairly straightforward, but stateful data systems in a distributed setup adds complexity. Hence it is recommented to keep stateful systems in a single node until scaling the system is costing high or high availability is required. 

- *Maintainability*: For better maintainability, we need to consider three design principles,
    - operability: make it easy for the operations team to keep the system running.
    - simplicity: make it easy to understand the system, by removing as much complexity as possible from the system using abstractions. Also not including any non-required complexities.
    - evolvability: make is easy to introduce changes to the system in future.
