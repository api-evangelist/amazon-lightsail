---
title: "Build high-performance apps with AWS Lambda Managed Instances"
url: "https://aws.amazon.com/blogs/compute/build-high-performance-apps-with-aws-lambda-managed-instances/"
date: "Mon, 30 Mar 2026 14:53:01 +0000"
author: "Debasis Rath"
feed_url: "https://aws.amazon.com/blogs/compute/feed/"
---
<p>High-performance applications such as CPU-intensive processing, memory-heavy analytics, and steady-state data pipelines often require more predictable compute resources than standard <a href="https://docs.aws.amazon.com/lambda/latest/dg/welcome.html" rel="noopener noreferrer" target="_blank">AWS Lambda</a> configurations provide. <a href="https://docs.aws.amazon.com/lambda/latest/dg/lambda-managed-instances.html" rel="noopener noreferrer" target="_blank">AWS Lambda Managed Instances (LMI)</a> addresses this by letting you run Lambda functions on selected Amazon EC2 <a href="https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-types.html" rel="noopener noreferrer" target="_blank">instance types</a> while preserving the Lambda programming model. You can choose over 400 Amazon Elastic Compute Cloud (Amazon EC2) instance types from general purpose, compute optimized, or memory optimized instance families to match workload requirements. AWS Lambda continues to manage infrastructure operations such as instance lifecycle management, operating system patching, runtime updates, request routing, and automatic scaling. This approach gives your teams greater control over compute characteristics, <a href="https://aws.amazon.com/ec2/pricing/" rel="noopener noreferrer" target="_blank">EC2 pricing model</a> and reduces operational overhead of managing servers or clusters.</p> 
<p>In this post, you will learn how to configure AWS Lambda Managed Instances by creating a <a href="https://docs.aws.amazon.com/lambda/latest/dg/lambda-managed-instances-capacity-providers.html" rel="noopener noreferrer" target="_blank">Capacity Provider</a> that defines your compute infrastructure, associating your Lambda function with that provider, and publishing a function version to provision the execution environments. We will conclude with production best practices including scaling strategies, thread safety, and observability for reliable performance.</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/25/create-lmi.png"><img alt="Figure 1. Creating Function on LMI" class="alignnone size-full wp-image-25941" height="467" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/25/create-lmi.png" width="1358" /></a></p> 
<p><em>Figure 1. Creating Function on LMI</em></p> 
<h2>Creating Capacity Providers</h2> 
<p>A Capacity Provider defines the infrastructure blueprint for running LMI functions on Amazon EC2. It specifies instance types, network placement, and scaling behavior. To create a Capacity Provider, you need two parameters: an IAM role (Capacity Provider Operator Role) granting Lambda permissions to launch and manage instances and your VPC configuration with subnets and security groups. Create this role in your account with the <code><a href="https://docs.aws.amazon.com/aws-managed-policy/latest/reference/AWSLambdaManagedEC2ResourceOperator.html" rel="noopener noreferrer" target="_blank">AWSLambdaManagedEC2ResourceOperator</a></code> managed policy following the <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html" rel="noopener noreferrer" target="_blank">Principle of Least Privilege</a> (granting only the minimum permissions necessary).</p> 
<p>This <a href="https://docs.aws.amazon.com/cli/latest/reference/lambda/create-capacity-provider.html" rel="noopener noreferrer" target="_blank">command</a> creates a Capacity Provider with instance types and scaling configuration:</p> 
<div class="hide-language"> 
 <pre><code class="lang-ruby">aws lambda create-capacity-provider \
  --capacity-provider-name my-lmi-capacity \
  --vpc-config SubnetIds=subnet-abc123,subnet-def456,SecurityGroupIds=sg-xyz789 \
  --permissions-config CapacityProviderOperatorRoleArn=arn:aws:iam::123456789012:role/LMIOperatorRole \
  --instance-requirements Architectures=x86_64,AllowedInstanceTypes=c5.2xlarge,r5.4xlarge \
  --capacity-provider-scaling-config MaxVCpuCount=50,ScalingMode=Auto \
  --region us-east-1</code></pre> 
</div> 
<p>This command returns a Capacity Provider ARN that you’ll use to create your LMI function. Your functions behavior depends on four main configurations in the capacity provider:</p> 
<h3>Instance selection</h3> 
<p>Lambda currently supports three Amazon EC2 instance families (.large and up): C (<a href="https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/compute-optimized-instances.html" rel="noopener noreferrer" target="_blank">compute optimized</a>) for CPU-heavy work, M (<a href="https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/general-purpose-instances.html" rel="noopener noreferrer" target="_blank">general purpose</a>) for balanced workloads, and R (<a href="https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/memory-optimized-instances.html" rel="noopener noreferrer" target="_blank">memory optimized</a>) for large datasets. Choose x86 (Intel/AMD) or ARM (Graviton) architectures. If you don’t specify instance types, Lambda defaults to appropriate instances based on your function’s memory and CPU configuration. This is the recommended starting point unless you have specific performance requirements. When you need more control, use <code>AllowedInstanceTypes</code> to specify only the instance types that Lambda can use or use <code>ExcludedInstanceTypes</code> to exclude specific types while allowing all other instance types. You can’t use both parameters together.</p> 
<h3>VPC and networking</h3> 
<p>Configure multiple subnets across Availability Zones. Lambda creates a minimum Amazon EC2 fleet of three instances distributed across your configured Availability Zones to maintain availability and resiliency. Egress traffic from functions, including Amazon CloudWatch Logs, transits through the Amazon EC2 instance’s network interface in your Amazon Virtual Private Cloud (Amazon VPC). As functions send logs and metrics to CloudWatch, you will need internet access through a NAT Gateway or <a href="https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints.html" rel="noopener noreferrer" target="_blank">VPC endpoints</a> with <a href="https://docs.aws.amazon.com/vpc/latest/privatelink/what-is-privatelink.html" rel="noopener noreferrer" target="_blank">AWS PrivateLink</a> for Amazon CloudWatch. This only affects egress traffic; function invoke requests don’t flow through your VPC. Security groups attached to your instances should allow only the traffic your function code needs. With LMI, configure VPC once at the Capacity Provider level instead of per function, simplifying management for multiple LMI functions. Standard Lambda functions continue to use their own VPC configurations. This Capacity Provider VPC configuration applies only to LMI functions.</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/25/image-2-11.png"><img alt="Figure 2. LMI Networking" class="alignnone size-full wp-image-25946" height="680" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/25/image-2-11.png" width="1543" /></a></p> 
<p><strong>Figure 2. LMI Networking</strong></p> 
<h3>Scaling configuration</h3> 
<p>Set <strong>MaxVCpuCount</strong> to cap compute capacity and control costs. New invocations throttle when you reach this limit until capacity frees up. Lambda monitors CPU utilization and scales instances automatically. Choose automatic scaling mode where Lambda tunes thresholds based on load patterns, or manual mode where you set a target CPU utilization percentage. Multiple functions can share the same Capacity Provider to reduce costs through better resource utilization, though you might want separate providers for functions with different performance or isolation requirements.</p> 
<h3>Security</h3> 
<p>Lambda encrypts <a href="https://docs.aws.amazon.com/ebs/latest/userguide/ebs-encryption.html" rel="noopener noreferrer" target="_blank">Amazon Elastic Block Store (Amazon EBS)</a> volumes attached to EC2 instances with a service-managed key by default. You can provide your own <a href="https://docs.aws.amazon.com/kms/latest/developerguide/overview.html" rel="noopener noreferrer" target="_blank">AWS Key Management Service (AWS KMS) key</a> for encryption. Place instances in private subnets with restrictive security groups for enhanced security.</p> 
<h2>Creating Lambda Managed Instance Functions</h2> 
<p>You create an LMI function similarly to creating a standard Lambda function. You package your code, set your runtime, assign an execution role, and configure memory. The difference is specifying a <code>CapacityProviderConfig</code> to tell Lambda which Capacity Provider to use and how to size each execution environment. Specify <code>CapacityProviderConfig</code> during function creation with the Capacity Provider ARN and configure two execution environment settings. <code>ExecutionEnvironmentMemoryGiBPerVCpu</code> sets the <code>memory-to-vCPU</code> ratio (2:1, 4:1, or 8:1) based on your workload type and <code>PerExecutionEnvironmentMaxConcurrency</code> defines how many concurrent requests share each execution environment. This table shows how memory and vCPU allocation maps across supported execution environment ratio.</p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <tbody> 
  <tr> 
   <td colspan="2" style="padding: 10px;">2:1 Ratio(Compute optimized)</td> 
   <td colspan="2" style="padding: 10px;">4:1 Ratio(General purpose)</td> 
   <td colspan="2" style="padding: 10px;">8:1 Ratio(Memory optimized)</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px;">Memory (GB)</td> 
   <td style="padding: 10px;">vCPU(s)</td> 
   <td style="padding: 10px;">Memory (GB)</td> 
   <td style="padding: 10px;">vCPU(s)</td> 
   <td style="padding: 10px;">Memory (GB)</td> 
   <td style="padding: 10px;">vCPU(s)</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px;">2</td> 
   <td style="padding: 10px;">1</td> 
   <td style="padding: 10px;">4</td> 
   <td style="padding: 10px;">1</td> 
   <td style="padding: 10px;">8</td> 
   <td style="padding: 10px;">1</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px;">4</td> 
   <td style="padding: 10px;">2</td> 
   <td style="padding: 10px;">8</td> 
   <td style="padding: 10px;">2</td> 
   <td style="padding: 10px;">16</td> 
   <td style="padding: 10px;">2</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px;">6</td> 
   <td style="padding: 10px;">3</td> 
   <td style="padding: 10px;">12</td> 
   <td style="padding: 10px;">3</td> 
   <td style="padding: 10px;">24</td> 
   <td style="padding: 10px;">3</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px;">8</td> 
   <td style="padding: 10px;">4</td> 
   <td style="padding: 10px;">16</td> 
   <td style="padding: 10px;">4</td> 
   <td style="padding: 10px;">32</td> 
   <td style="padding: 10px;">4</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px;">10</td> 
   <td style="padding: 10px;">5</td> 
   <td style="padding: 10px;">20</td> 
   <td style="padding: 10px;">5</td> 
   <td style="padding: 10px;"></td> 
   <td style="padding: 10px;"></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px;">12</td> 
   <td style="padding: 10px;">6</td> 
   <td style="padding: 10px;">24</td> 
   <td style="padding: 10px;">6</td> 
   <td style="padding: 10px;"></td> 
   <td style="padding: 10px;"></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px;">14</td> 
   <td style="padding: 10px;">7</td> 
   <td style="padding: 10px;">28</td> 
   <td style="padding: 10px;">7</td> 
   <td style="padding: 10px;"></td> 
   <td style="padding: 10px;"></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px;">16</td> 
   <td style="padding: 10px;">8</td> 
   <td style="padding: 10px;">32</td> 
   <td style="padding: 10px;">8</td> 
   <td style="padding: 10px;"></td> 
   <td style="padding: 10px;"></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px;">…</td> 
   <td style="padding: 10px;">…</td> 
   <td style="padding: 10px;"></td> 
   <td style="padding: 10px;"></td> 
   <td style="padding: 10px;"></td> 
   <td style="padding: 10px;"></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px;">32</td> 
   <td style="padding: 10px;">16</td> 
   <td style="padding: 10px;"></td> 
   <td style="padding: 10px;"></td> 
   <td style="padding: 10px;"></td> 
   <td style="padding: 10px;"></td> 
  </tr> 
 </tbody> 
</table> 
<h3>Function Memory-to-CPU configuration</h3> 
<p>Set the function’s memory size (up to 32 GB for LMI) and <code>ExecutionEnvironmentMemoryGiBPerVCpu</code> ratio. The default ratio is 2:1. A 2:1 ratio map to compute optimized instances for CPU-intensive tasks like video encoding, 4:1 map to a general purpose for balanced workloads, and 8:1 maps to a memory optimized instances for large in-memory datasets or caching. You must set memory in multiples of the ratio. LMI requires a 2 GB minimum as execution environments need sufficient memory to handle multiple concurrent requests. LMI supports up to 32 GB memory per execution environment.</p> 
<h3>Multi-Concurrency settings</h3> 
<p>LMI supports <a href="https://docs.aws.amazon.com/lambda/latest/dg/lambda-managed-instances-runtimes.html" rel="noopener noreferrer" target="_blank">multiple concurrent invocations</a> sharing the same execution environment, reducing cost per invocation by maximizing vCPU utilization. This is particularly effective for I/O-bound workloads, where invocations waiting on database queries or API calls yield vCPU usage to other invocations during idle periods. Lambda defaults to max concurrency per execution environment based on your runtime: Node.js (64 per vCPU), Java, and .NET (32 per vCPU), Python (16 per vCPU). Use <code>PerExecutionEnvironmentMaxConcurrency</code> to set a lower limit based on your workload’s resource needs. Decrease it if you’re experiencing memory pressure or CPU contention. When environments reach their configured max concurrency, new invocations throttle until capacity frees up at the execution environment level. This table captures the maximum concurrency per vCPU for each supported programming language.</p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <tbody> 
  <tr> 
   <td style="padding: 10px;">Language</td> 
   <td style="padding: 10px;">Default Max Concurrency</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px;">Node.js</td> 
   <td style="padding: 10px;">64 per vCPU</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px;">Java</td> 
   <td style="padding: 10px;">32 per vCPU</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px;">.NET</td> 
   <td style="padding: 10px;">32 per vCPU</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px;">Python</td> 
   <td style="padding: 10px;">16 per vCPU</td> 
  </tr> 
 </tbody> 
</table> 
<p>This <a href="https://docs.aws.amazon.com/cli/latest/reference/lambda/create-function.html" rel="noopener noreferrer" target="_blank">command</a> creates a Lambda function and associates it with your Capacity Provider:</p> 
<div class="hide-language"> 
 <pre><code class="lang-javascript">aws lambda create-function \
  --function-name my-lmi-function \
  --runtime python3.13 \
  --role arn:aws:iam::123456789012:role/LambdaExecutionRole \
  --handler app.lambda_handler \
  --zip-file fileb://function.zip \
  --memory-size 4096 \
  --capacity-provider-config '{
    "LambdaManagedInstancesCapacityProviderConfig": {
      "CapacityProviderArn": "arn:aws:lambda:us-east-1:123456789012:capacity-provider:my-lmi-capacity",
      "ExecutionEnvironmentMemoryGiBPerVCpu": 4.0,
      "PerExecutionEnvironmentMaxConcurrency": 10
    }
  }' \
  --region us-east-1</code></pre> 
</div> 
<h2>Publishing Lambda Managed Instance Functions</h2> 
<p><strong>Important:</strong>&nbsp;publish a function version before invoking an LMI function. Publishing triggers Lambda to provision Amazon EC2 instances and initialize execution environments, so that the configured baseline capacity is ready before you start invoking. Expect a brief delay before your code goes live as Lambda provisions and launches Amazon EC2 instances. With LMI, execution environments pre-warm after publishing and remain invoke-ready, without cold starts for published versions. Standard Lambda environments initialize on first invoke (cold starts).</p> 
<p>This <a href="https://docs.aws.amazon.com/cli/latest/reference/lambda/publish-version.html" rel="noopener noreferrer" target="_blank">command</a> publishes a Lambda function version and provisions capacity:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">aws lambda publish-version --function-name my-lmi-function \
--region us-east-1</code></pre> 
</div> 
<p>After publishing, the function works with standard invocation methods including direct invokes, <a href="https://docs.aws.amazon.com/lambda/latest/dg/invocation-eventsourcemapping.html" rel="noopener noreferrer" target="_blank">event source mappings</a>, and service integrations with Amazon API Gateway, Amazon Simple Storage Service (Amazon S3), Amazon DynamoDB Streams, and Amazon EventBridge.</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/25/image-3-8.png"><img alt="Figure 3. LMI Invocation from event sources" class="alignnone size-full wp-image-25947" height="519" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/25/image-3-8.png" width="1073" /></a></p> 
<p><em>Figure 3. LMI Invocation from event sources</em></p> 
<h2>Scaling LMI Functions</h2> 
<p>Lambda monitors <a href="https://docs.aws.amazon.com/lambda/latest/dg/lambda-managed-instances-scaling.html" rel="noopener noreferrer" target="_blank">CPU utilization</a> at Capacity Provider level. When CPU utilization reaches the target threshold, Lambda automatically provisions additional EC2 instances, and creates more execution environments on those instances, up to the <code>MaxVCpuCount</code> limit you configured for your capacity provider. As demand decreases, Lambda consolidates workloads onto fewer EC2 instances. You can choose <a href="https://docs.aws.amazon.com/lambda/latest/dg/lambda-managed-instances-scaling.html" rel="noopener noreferrer" target="_blank">automatic scaling mode</a> (Lambda adjusts thresholds based on your patterns) or manual mode (you set a target CPU percentage). Automatic mode works for variable traffic patterns or when getting started. Manual mode fits when you have predictable patterns and want precise control over scaling thresholds for cost optimization.</p> 
<h3>Min and max execution environments</h3> 
<p>Control scaling at the function level with min and max execution environments. The default minimum is 3 execution environments to maintain <a href="https://docs.aws.amazon.com/lambda/latest/dg/lambda-concurrency.html" rel="noopener noreferrer" target="_blank">high availability</a> across Availability Zones. Your total function concurrency equals the number of execution environments multiplied by <code>PerExecutionEnvironmentMaxConcurrency</code>. For example, with min set to 3 and <code>PerExecutionEnvironmentMaxConcurrency</code> of 10, you have provided capacity for 30 concurrent invocations. With max set to 20, you can scale up to 200 concurrent invocations with incoming traffic, based on CPU utilization or concurrency saturation per execution environment. Set max to cap total concurrency and prevent noisy neighbor issues when multiple functions share a Capacity Provider. LMI maintains a minimum number of execution environments with a minimum Amazon EC2 fleet, while standard Lambda scales to zero when idle. Set both min and max to 0 to deactivate a function without deleting it.</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/25/image-4-7.png"><img alt="Figure 4. LMI Scaling" class="alignnone size-full wp-image-25936" height="615" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/25/image-4-7.png" width="1241" /></a></p> 
<p><strong>Figure 4. LMI Scaling</strong></p> 
<p>This command updates the minimum and maximum execution environments for your function:</p> 
<div class="hide-language"> 
 <pre><code class="lang-javascript">aws lambda put-function-scaling-config \
  --function-name my-lmi-function \
  --qualifier $LATEST \
  --function-scaling-config MinExecutionEnvironments=5,MaxExecutionEnvironments=20 \
  --region us-east-1</code></pre> 
</div> 
<p>We’ll cover scaling patterns and throughput optimization strategies in depth in a separate blog post.</p> 
<h2>Best Practices and Production Considerations</h2> 
<h3>Thread Safety</h3> 
<p>Since LMI supports multiple invocations sharing execution environments, your code must be <a href="https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html" rel="noopener noreferrer" target="_blank">thread-safe.</a> Code that isn’t thread-safe causes data corruption, security issues, or unpredictable behavior under concurrent load.</p> 
<h4>Thread safety essentials</h4> 
<p>Avoid mutating shared objects or global variables. Use thread-local storage for request-specific data. Initialize shared clients (AWS SDK, database connections) outside the function handler and verify that configurations remain immutable during invocations. Write to <code>/tmp</code> using request-specific file names to prevent concurrent writes.</p> 
<h4>Runtime-specific guidance</h4> 
<p>Java applications should use immutable objects, thread-safe collections, and proper synchronization. Node.js applications should use async context for request isolation. Python applications run separate processes per execution environment. So, focus on interprocess coordination and file locking for <code>/tmp</code> access.</p> 
<h3>Workload Optimization</h3> 
<p>I/O-bound workloads perform better with higher concurrency per environment. Use asynchronous patterns and non-blocking I/O to maximize efficiency. CPU-bound workloads get no benefit from concurrency greater than one per vCPU. Instead, configure more vCPUs per function for true parallelism for compute-heavy tasks like data transformation or image processing.</p> 
<h3>Testing</h3> 
<p>Validate your code under concurrent execution. Test with multiple simultaneous invocations to detect race conditions and shared state issues before production deployment. You can use LocalStack for local emulation of LMI. Learn more about LocalStack’s LMI support in their <a href="https://blog.localstack.cloud/testing-locally-with-lambda-managed-instances/" rel="noopener noreferrer" target="_blank">announcement blog</a>.</p> 
<h3>Compatibility</h3> 
<p>Tools like <a href="https://docs.aws.amazon.com/powertools/" rel="noopener noreferrer" target="_blank">Powertools</a> for AWS work with LMI without code changes. However, if you’re reusing existing Lambda function code, layers, or packaged dependencies on LMI, test for thread safety and compatibility with the multi-concurrent execution model before production deployment.</p> 
<h3>Observability</h3> 
<p>LMI automatically publishes CloudWatch metrics at two levels: capacity provider (CPU, memory, network, and disk utilization across your Amazon EC2 fleet) and execution environment (concurrency, CPU, and memory per function). Monitor <code>CPUUtilization</code> to understand scaling headroom and right-size your <code>MaxVCpuCount</code>. Track <code>ExecutionEnvironmentConcurrency</code> against <code>ExecutionEnvironmentConcurrencyLimit</code> to catch throttling before it impacts users. Lambda publishes metrics at 5-minute intervals. Use CloudWatch alarms to stay ahead of capacity limits in production.</p> 
<h2>Conclusion</h2> 
<p>AWS Lambda Managed Instances combines serverless simplicity with compute flexibility, helping you run high-performance workloads with reduced operational complexity. You maintain the familiar programming model of Lambda while accessing the diverse instance types of Amazon EC2 and predictable pricing, making it well-suited for data processing pipelines, compute intensive operations and cost-sensitive steady-state applications.</p> 
<p>Ready to get started with LMI?&nbsp;Deploy our&nbsp;<a href="https://github.com/aws-samples/sample-aws-lambda-managed-instances/tree/main/examples/fsi/sample-retirement-savings-simulator" rel="noopener noreferrer" target="_blank">Monte Carlo risk simulation example&nbsp;</a>from GitHub to see LMI in action with a real compute-intensive workload. The sample includes complete infrastructure code and walks you through capacity provider configuration, function setup, and performance optimization.</p> 
<p>We want to hear from you. Share your feedback, questions, and use cases on <a href="https://repost.aws/" rel="noopener noreferrer" target="_blank">re:Post</a>.</p>
