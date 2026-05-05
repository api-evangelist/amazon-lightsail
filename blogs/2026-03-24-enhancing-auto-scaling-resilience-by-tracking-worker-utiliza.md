---
title: "Enhancing auto scaling resilience by tracking worker utilization metrics"
url: "https://aws.amazon.com/blogs/compute/enhancing-auto-scaling-resilience-by-tracking-worker-utilization-metrics/"
date: "Tue, 24 Mar 2026 16:17:58 +0000"
author: "Brian Moore"
feed_url: "https://aws.amazon.com/blogs/compute/feed/"
---
<p>A resilient auto scaling policy requires metrics that correlate with application utilization, which may not be tied to system resources. Traditionally, auto scaling policies track system resource such as CPU utilization. These metrics are easily available, but they only work when resource consumption correlates with worker capacity. Factors such as high variance in request processing time, mixed instance types, or natural changes in application behavior over time can break this assumption.</p> 
<p>Worker utilization tracking offers an alternative approach. Using a combination of total worker slots, work in flight, and work waiting in the backlog, a utilization value can be calculated for use in an auto scaling policy. This approach remains accurate across fleets with mixed instance types, applications with variable latencies, and requires no changes as your application evolves.</p> 
<h2>The limitations of resource-based scaling</h2> 
<p>Traditional auto scaling policies track system resource metrics like CPU utilization, assuming a direct correlation between resource consumption and available application capacity. Consider an application that reads messages from <a href="https://aws.amazon.com/sqs/" rel="noopener noreferrer" target="_blank">Amazon Simple Queue Service (SQS)</a>, processes them, and writes results to <a href="https://aws.amazon.com/dynamodb/" rel="noopener noreferrer" target="_blank">Amazon DynamoDB</a>. If this application uses a fixed-size thread pool to process messages, such as 10 worker threads, the application reaches maximum capacity when all threads are busy, regardless of CPU utilization.</p> 
<p>In our example, each worker spends most of its time waiting for DynamoDB responses rather than consuming CPU. All 10 threads become occupied handling requests, but CPU utilization stays low. From the perspective of the auto scaling policy, the fleet looks like it has enough capacity because plenty of CPU headroom remains. Meanwhile, new messages accumulate in the SQS queue because no workers are available to process them.</p> 
<p>For queue-based workloads, <a href="https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-using-sqs-queue.html#scale-sqs-queue-custom-metric" rel="noopener noreferrer" target="_blank">AWS provides guidance</a> to scale based on an acceptable backlog per worker. This is a calculated target based on your application’s average processing latency (queue delay). This works well when processing times are consistent, but breaks down if an application has variable latency characteristics.</p> 
<p>Consider an image processing application that initially handles thumbnails taking 500 ms each. Using the traditional guidance with a target latency of 5 seconds you calculate an acceptable backlog of 10 messages per worker and deploy your scaling policy. Over time, the application evolves to also process 4K photos which take 2 seconds each. Eventually 4K photos are 50% of your traffic and total latency for queued messages has increased to 12.5 seconds, 2.5x more than your initial target.</p> 
<p>The scaling policy is no longer fit for its intended purpose because your original latency assumptions no longer reflect reality. To keep this type of scaling effective you must also remember to update your scaling policies as your application behavior evolves.</p> 
<p>A shift to using mixed instance types in your application can lead to additional complexity when using traditional resource-based scaling policies. Different instance types may handle the same workload at different CPU levels leading to an unbalanced average that misrepresents your actual application health. By changing your mental model to consider how much work your application can accept instead of how much of a system resource is available you can improve your scaling rules and better model your application’s capacity.</p> 
<h2>Understanding worker utilization</h2> 
<p>Worker utilization measures the ratio of active work to available processing capacity. To calculate it, divide total work by total workers.</p> 
<p>We use an SQS-based processing application as an example to demonstrate how worker utilization operates, but this approach can also be applied to other applications where work units and worker capacity are measurable. In our example application total work consists of messages waiting to be processed plus messages currently being processed. <a href="https://aws.amazon.com/cloudwatch/" rel="noopener noreferrer" target="_blank">Amazon CloudWatch</a> provides these values through the <code>ApproximateNumberOfMessagesVisible</code> metric (messages waiting in the queue) and the <code>ApproximateNumberOfMessagesNotVisible</code> metric (messages currently being processed or in flight). Each host in your application should publish the number of available workers as a custom CloudWatch metric with at least a 1-minute period. For Java thread pools or Python multiprocessing pools, this represents the pool or process count. The formula works regardless of the metric period. Using the shortest period possible allows more responsive target tracking and enables <a href="https://aws.amazon.com/blogs/compute/faster-scaling-with-amazon-ec2-auto-scaling-target-tracking/" rel="noopener noreferrer" target="_blank">Fast Target Tracking</a> if your application has sub-minute data points.</p> 
<p>To derive the formula, we can use the following CloudWatch Metric Math expressions:</p> 
<ul> 
 <li><code>totalWork</code> = FILL(<code>backlog</code>, REPEAT) + FILL(<code>inFlight</code>, REPEAT)</li> 
 <li><code>utilizationRatio</code> = <code>totalWork</code> / <code>workers</code></li> 
</ul> 
<p>Where:</p> 
<ul> 
 <li><code>backlog</code> = <code>ApproximateNumberOfMessagesVisible</code> with the <code>Maximum</code> statistic.</li> 
 <li><code>inFlight</code> = <code>ApproximateNumberOfMessagesNotVisible</code> with the <code>Maximum</code> statistic.</li> 
 <li><code>workers</code> = Your custom <code>TotalWorkers</code> metric with the <code>Sum</code> statistic.</li> 
</ul> 
<p>Putting the components together the final expression for your target tracking scaling policy uses the following formula:</p> 
<p>IF(FILL(<code>workers</code>, 0) &gt; 0, <code>utilizationRatio</code>, IF(<code>totalWork</code> &gt; 0, 1, 0))</p> 
<p>The FILL function uses last known values if SQS metrics are delayed, and the IF statement handles the case where you have no traffic and your fleet scales to zero instances. When there are no available workers, the formula metric reports 1 to indicate that the workers are fully saturated. This prevents the application from getting stuck at zero capacity and not being able to respond to any requests.</p> 
<p>In this formula, a value of 1 or higher represents full or over saturation, where all workers are busy with no spare capacity, like running at 100% CPU. Values below 1 indicate available capacity for your application to process more work.</p> 
<p>For applications without a measurable backlog metric, you can track worker utilization using only the in-flight work. This approach works for APIs or other synchronous workloads where work arrives and is immediately assigned to workers rather than queuing. In these cases, the formula becomes:</p> 
<p>IF(FILL (<code>workers</code>, 0) &gt; 0, <code>utilizationRatio</code>, IF(FILL(<code>inFlight</code>, 0) &gt; 0, 1, 0))</p> 
<p>In this scenario the utilization ratio is calculated as follows:</p> 
<ul> 
 <li><code>utilizationRatio</code> = FILL(<code>inFlight</code>, REPEAT) / <code>workers</code></li> 
</ul> 
<p>The definitions of <code>workers</code> and <code>inFlight</code> remain the same for this formula. The primary difference is that the ratio directly tracks workers available and does not consider the backlog as an option.</p> 
<h2>How worker utilization prevents outages</h2> 
<p>Worker utilization-based scaling works for any application that can define available workers and total work. When the ratio of total work to available workers exceeds your threshold, the system scales out. This approach measures whether workers are available to handle the workload and treats application bottlenecks consistently. Whether workers are waiting on network I/O, performing CPU-intensive calculations, or experiencing another bottleneck doesn’t matter; the only question is whether total work exceeds available worker capacity. Any situation causing messages to accumulate on the queue increases the utilization ratio and triggers scale-out.</p> 
<h2>Implementing worker utilization scaling</h2> 
<p>To set up worker utilization-based auto scaling, identify metrics to use in the formula discussed earlier. First, identify a metric to track the amount of work being worked on. For SQS-based processing, AWS provides this metric. Second, implement a custom metric from your application representing the total workers. Optionally you can also identify a metric to track the available backlog of work.</p> 
<p>Using CloudWatch metric math, you calculate the utilization metric and use it in a target tracking scaling policy. Here is an example <a href="https://aws.amazon.com/cloudformation/" rel="noopener noreferrer" target="_blank">AWS CloudFormation</a> snippet showing the metric math configuration for a <a href="https://aws.amazon.com/pm/ec2/" rel="noopener noreferrer" target="_blank">Amazon EC2</a> Auto Scaling group. This snippet shows only the scaling policy configuration and is only an example, before using in production fully test with your application. Your complete template also needs IAM roles with appropriate permissions for SQS, DynamoDB, and CloudWatch access.</p> 
<pre><code class="lang-yaml">ScalingPolicy: 
  Type: AWS::AutoScaling::ScalingPolicy 
  Properties: 
    AutoScalingGroupName: !Ref AutoScalingGroup 
    PolicyType: TargetTrackingScaling 
    TargetTrackingConfiguration: 
      TargetValue: 0.7 
      CustomizedMetricSpecification: 
        Metrics: 
          - Id: backlog 
            MetricStat: 
            Metric: 
              Namespace: AWS/SQS 
              MetricName: ApproximateNumberOfMessagesVisible 
              Dimensions: 
                - Name: QueueName 
                  Value: !GetAtt ProcessingQueue.QueueName 
              Stat: Maximum 
          - Id: inFlight 
            MetricStat: 
            Metric: 
              Namespace: AWS/SQS 
              MetricName: ApproximateNumberOfMessagesNotVisible 
              Dimensions: 
                - Name: QueueName 
                  Value: !GetAtt ProcessingQueue.QueueName 
              Stat: Maximum 
          - Id: workers 
            MetricStat: 
            Metric: 
              Namespace: YourApp 
              MetricName: TotalWorkers 
            Stat: Sum 
          - Id: totalWork 
            Expression: FILL(backlog, REPEAT) + FILL(inFlight, REPEAT) 
          - Id: utilizationRatio 
            Expression: totalWork / workers 
          - Id: utilization 
            Expression: IF(FILL(workers, 0) &gt; 0, utilizationRatio, IF(totalWork &gt; 0, 1, 0)) 
            ReturnData: true</code></pre> 
<p>This approach also works for <a href="https://aws.amazon.com/ecs/" rel="noopener noreferrer" target="_blank">Amazon ECS</a> services using <a href="https://aws.amazon.com/autoscaling/" rel="noopener noreferrer" target="_blank">AWS Application Auto Scaling</a>. The metric math configuration remains the same, but you create an <code>AWS::ApplicationAutoScaling::ScalingPolicy</code> resource instead, adapting the parameters accordingly.</p> 
<h2>Choosing a target utilization</h2> 
<p>Since the worker utilization metric directly tracks the available capacity of your application, the target utilization value you choose reflects your organization’s balance between cost efficiency and availability. Lower target values provide more headroom for traffic spikes and faster response to load changes but result in higher infrastructure costs due to lower utilization. Higher target values maximize cost efficiency by keeping workers busy but leave less headroom for sudden traffic increases.</p> 
<p>When choosing a target consider traffic patterns, acceptable latency during scale-out events, and cost sensitivity. Applications with unpredictable traffic spikes may benefit from lower targets, while an application with predictable load can safely use higher targets. Start with a moderate value like 0.7 and adjust based on observed behavior and your business requirements. If you previously tracked a resource utilization metric such as CPU, consider starting with the same target.</p> 
<h2>Monitoring resource utilization for cost optimization</h2> 
<p>While worker utilization drives scaling decisions, CPU and latency should be regularly evaluated to ensure cost-effective operations. Resource-based metrics can identify host resizing opportunities to better match your application requirements. If no scale-in happens when CPU utilization is consistently low, you are likely running instances that are too large for your workload. By using worker utilization in an auto scaling policy, you can switch to a different instance type without adjusting the auto scaling policy. The formula automatically adapts as you add different instance types or update the capacity per worker.</p> 
<p>Conversely, if CPU utilization is consistently high while worker utilization remains at your target, your instances might be undersized. Upgrading to larger instance types can improve per-worker throughput, allowing each worker to process tasks faster. Changes to your auto scaling policy are not needed in this situation either. As messages are processed faster, they spend less time in the in-flight state, and the utilization ratio naturally adjusts.</p> 
<p>This approach manages application availability independent of instance size, while resource utilization guides cost optimization. Each can be optimized independently without complex coordination.</p> 
<h2>Conclusion</h2> 
<p>Worker utilization-based auto scaling reduces the operational burden of continuously validating your scaling rules as application requirements and infrastructure change. By tracking the ratio of work to workers, your auto scaling policies automatically respond to capacity constraints based on available work. The approach works across workloads with discrete processing units and remains effective when you modify instance configurations or application worker pool sizes.</p> 
<p>Implementation requires identifying a metric for available work, publishing a custom metric representing total workers, and using CloudWatch metric math in a target tracking scaling policy. This setup provides resilience that scaling based solely on resource metrics cannot achieve, while maintaining the flexibility to optimize costs and change your instance size without impacting system availability.</p> 
<p>To get started:</p> 
<ol> 
 <li>Identify an application in your environment that uses a worker pool.</li> 
 <li>Instrument the application to publish worker count metrics.</li> 
 <li>Configure a scaling policy tracking worker utilization.</li> 
 <li>Monitor how the system responds to traffic changes and capacity events.</li> 
</ol> 
<h2>Learn more</h2> 
<p>To learn more about auto scaling and monitoring, see the following resources:</p> 
<ul> 
 <li><a href="https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-scaling-target-tracking.html" rel="noopener noreferrer" target="_blank">Amazon EC2 Auto Scaling target tracking scaling policies</a></li> 
 <li><a href="https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-autoscaling-targettracking.html" rel="noopener noreferrer" target="_blank">AWS Application Auto Scaling for Amazon ECS services</a></li> 
 <li><a href="https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/using-metric-math.html" rel="noopener noreferrer" target="_blank">Using Amazon CloudWatch metric math</a></li> 
 <li><a href="https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/publishingMetrics.html" rel="noopener noreferrer" target="_blank">Publishing custom CloudWatch metrics</a></li> 
</ul>
