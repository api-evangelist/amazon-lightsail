---
title: "Building Memory-Intensive Apps with AWS Lambda Managed Instances"
url: "https://aws.amazon.com/blogs/compute/building-memory-intensive-apps-with-aws-lambda-managed-instances/"
date: "Fri, 10 Apr 2026 19:54:44 +0000"
author: "Guy Haddad"
feed_url: "https://aws.amazon.com/blogs/compute/feed/"
---
<p>Building memory-intensive applications with <a href="https://aws.amazon.com/lambda/" rel="noopener noreferrer" target="_blank">AWS Lambda</a> just got easier. <a href="https://aws.amazon.com/lambda/lambda-managed-instances/" rel="noopener noreferrer" target="_blank">AWS Lambda Managed Instances</a> gives you up to 32 GB of memory—3x more than standard AWS Lambda—while maintaining the serverless experience you know. Modern applications increasingly require substantial memory resources to process large datasets, perform complex analytics, and deliver real-time insights for use cases such as in-memory analytics, Machine Learning (ML) model inference, and real-time semantic search. AWS Lambda Managed Instances gives you a familiar serverless programming model and experience combined with the flexibility of being able to choose the underlying <a href="https://aws.amazon.com/ec2/" rel="noopener noreferrer" target="_blank">Amazon EC2</a> instance types and providing developers with access to large memory configurations.</p> 
<p>In this post, you will see how AWS Lambda Managed Instances enables memory-intensive workloads that were previously challenging to run in serverless environments, using an AI-powered customer analytics application as a practical example. You’ll see cost savings of up to 33% compared to standard Lambda for predictable workloads, while eliminating the operational overhead of managing EC2 instances.</p> 
<h2><strong>Understanding AWS Lambda Managed Instances</strong></h2> 
<p>AWS Lambda Managed Instances runs your AWS Lambda functions on the Amazon EC2 instance types of your choice in your account, including <a href="https://aws.amazon.com/ec2/graviton/" rel="noopener noreferrer" target="_blank">Graviton4</a> and memory-optimized instance types. AWS handles underlying infrastructure lifecycle including provisioning, scaling, patching, and routing, while you benefit from Amazon EC2 pricing advantages like <a href="https://aws.amazon.com/savingsplans/" rel="noopener noreferrer" target="_blank">Savings Plans</a> and <a href="https://aws.amazon.com/aws-cost-management/aws-cost-optimization/reserved-instances/" rel="noopener noreferrer" target="_blank">Reserved Instances</a>.</p> 
<p><strong>Key benefits include:</strong></p> 
<ul> 
 <li><strong>Flexible instance selection:</strong> Choose from compute-optimized (C), general-purpose (M), and memory-optimized (R) instance families</li> 
 <li><strong>Configurable memory-CPU ratios:</strong> Optimize resource allocation for your workload</li> 
 <li><strong>Multi-concurrent invocations:</strong> One execution environment handles multiple invocations simultaneously, improving utilization for I/O-heavy applications</li> 
 <li><strong>Dynamic scaling:</strong> Instances scale based on CPU utilization without cold starts</li> 
</ul> 
<p>AWS Lambda Managed Instances is best suited for high-volume, predictable workloads that benefit from sustained compute capacity and larger memory configurations.</p> 
<h2><strong>Memory-Intensive Workloads Work Best with AWS Lambda Managed Instances</strong></h2> 
<p>This blog focuses on one of AWS Lambda Managed Instances’ most powerful capabilities: running memory-intensive workloads that require more than the standard AWS Lambda’s 10 GB memory and 250MB ZIP limits. Here are the use cases where AWS Lambda Managed Instances helps:</p> 
<ul> 
 <li><strong>In-Memory Analytics</strong> — Load gigabytes of structured data into memory at initialization and serve sub-millisecond analytical queries across thousands of invocations</li> 
 <li><strong>ML Model Inference</strong> — Keep large model weights resident in memory across invocations for consistent, low-latency inference without a dedicated endpoint.</li> 
 <li><strong>Real-Time Semantic Search</strong> — Build vector similarity search over large embedding indexes held entirely in memory, enabling natural language queries over millions of records without an external vector database.</li> 
 <li><strong>Graph Processing</strong> — Hold large graph structures in memory for traversal algorithms that require the full graph to be accessible at once.</li> 
 <li><strong>Scientific &amp; Numerical Computing</strong> — Run simulations, Monte Carlo methods, and large matrix operations that require substantial working memory and benefit from memory-optimized Amazon EC2 instance families.</li> 
 <li><strong>Large-Scale Report Generation</strong> — Aggregate and transform multi-gigabyte datasets in memory to generate complex reports or dashboards on demand, without staging data through intermediate storage.</li> 
</ul> 
<h2><strong>Use Case: AI-Powered Customer Analytics with AWS Lambda Managed Instances</strong></h2> 
<p>To demonstrate the power of AWS Lambda Managed Instances for memory-intensive applications, we built an AI-Powered Customer Analytics application that combines in-memory data processing with ML-based semantic search. The application loads in memory 1 million customer behavioral records (sessions, purchases, browsing patterns) from a Parquet file in S3 into a Pandas DataFrame and an embeddings cache consuming 200MB, then responds for analytics queries:</p> 
<ol> 
 <li><strong>Customer Analysis</strong> — Deep-dive into individual customer behavior: engagement scores, conversion rates, purchase patterns, and AI-generated customer segments</li> 
 <li><strong>Semantic Search</strong> — Natural language queries powered by FastEmbed (sentence-transformers/all-MiniLM-L6-v2) that find similar customers using vector similarity</li> 
 <li><strong>Cohort Analysis</strong> — Real-time segmentation by device, country, age group with aggregated metrics</li> 
</ol> 
<h3><strong>Architecture Overview</strong></h3> 
<p>Our AI-powered customer analytics application demonstrates this in practice: 1 million records in memory (200MB), a compact sentence transformer model for semantic search, sub-second query performance, and zero infrastructure to manage. The solution uses a simple, serverless architecture:</p> 
<ul> 
 <li>Customer transaction data (Parquet format) is stored in Amazon S3</li> 
 <li>Amazon Cognito User Pool authenticates users and issues JWT tokens for API access</li> 
 <li>Amazon API Gateway routes requests with Cognito authorizer validation, rate limiting (5 requests/second, burst 10), X-Ray tracing, and access logging</li> 
 <li>AWS Lambda function with AWS Lambda Managed Instances loads the entire dataset (200MB) and all-MiniLM-L6-v2 model (900MB) into memory during initialization while also performing a threaded embeddings cache generation. This step can consume about 14GB of the allocated memory, exceeding standard AWS Lambda’s 10 GB limit</li> 
 <li>Analytics queries execute against the in-memory data using the model</li> 
 <li>Results are returned in milliseconds for interactive analysis</li> 
</ul> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/04/09/imageComputeBlog-2543-1.png"><img alt="Architecture diagram" class="alignnone size-full wp-image-26050" height="718" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/04/09/imageComputeBlog-2543-1.png" width="1566" /></a></p> 
<h3><strong>Deploy the Application</strong></h3> 
<p>The below steps walk you through deploying the application to AWS using the AWS Serverless Application Model (SAM). The deployment process packages your Lambda function code, uploads artifacts to Amazon S3, and provisions all required AWS resources including Lambda functions, IAM roles, and any configured VPC networking via AWS CloudFormation.</p> 
<h3><strong>Prerequisites</strong></h3> 
<p>Make sure you have the following tools installed locally:</p> 
<ul> 
 <li><a href="https://aws.amazon.com/cli/" rel="noopener noreferrer" target="_blank">AWS CLI</a> configured with credentials</li> 
 <li><a href="https://aws.amazon.com/serverless/sam/" rel="noopener noreferrer" target="_blank">SAM CLI</a> installed</li> 
 <li>Python 3.13+ installed locally</li> 
 <li><a href="https://www.docker.com/" rel="noopener noreferrer" target="_blank">Docker</a> or <a href="https://runfinch.com/" rel="noopener noreferrer" target="_blank">Finch</a> (required for container builds)</li> 
 <li>AWS account with appropriate permissions</li> 
 <li>A VPC with at least 2 subnets (across different Availability Zones) and a security group — required for the Lambda Managed Instances capacity provider</li> 
 <li>Supported regions: Check <a href="https://builder.aws.com/capabilities/" rel="noopener noreferrer" target="_blank">AWS Capabilities by Region</a> for supported regions</li> 
</ul> 
<h2><strong>Getting Started</strong></h2> 
<p>The complete source code for this application is available in our GitHub repository. To deploy it yourself follow the below steps and refer to the full <a href="https://github.com/aws-samples/sample-lambda-managed-instances-analytics">deployment instructions</a> hosted on GitHub.</p> 
<p><strong>1. Clone the repository</strong></p> 
<p><em>git clone <a href="https://github.com/aws-samples/sample-lambda-managed-instances-analytics.git">https://github.com/aws-samples/sample-lambda-managed-instances-analytics.git</a></em></p> 
<p><strong>2. Navigate to the project folder</strong></p> 
<p><em>cd sample-lambda-managed-instances-analytics</em></p> 
<p><em>chmod +x setup-data.sh deploy-lambda.sh</em></p> 
<p><strong>3. Generate sample data and upload to S3</strong></p> 
<p><em>./setup-data.sh</em></p> 
<p>This script will create an S3 bucket (if needed), generate 1M rows of sample data, and upload the data to S3.</p> 
<p><strong>4. Build and deploy the Lambda function</strong></p> 
<p><em>./deploy-lambda.sh</em></p> 
<p>This script will build the container image with FastEmbed, push it to ECR, and deploy the Lambda function along with Capacity Provider, API Gateway, and Cognito User Pool. After deployment, it automatically generates the UI authentication configuration and prompts you to create a test user.</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/04/09/imageComputeBlog-2543-2.png"><img alt="SAM template" class="alignnone size-full wp-image-26051" height="221" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/04/09/imageComputeBlog-2543-2.png" width="484" /></a></p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/04/09/imageComputeBlog-2543-3.png"><img alt="Capacity provider configuration" class="alignnone size-full wp-image-26052" height="430" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/04/09/imageComputeBlog-2543-3.png" width="1071" /></a></p> 
<h3><strong>Run the Application</strong></h3> 
<p><strong>1. Start the UI</strong></p> 
<p>The application includes a simple HTML-based UI through which you can test the AWS Lambda function using Amazon API Gateway:</p> 
<p><em>cd ui &amp;&amp; python3 -m http.server 8000</em></p> 
<p>2. Open your browser at <a href="http://localhost:8000" rel="noopener noreferrer" target="_blank">http://localhost:8000</a> and click ‘Sign In’ to authenticate via Cognito using the username/password that you created during deployment</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/04/09/imageComputeBlog-2543-4.png"><img alt="Starting the UI" class="alignnone size-full wp-image-26053" height="256" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/04/09/imageComputeBlog-2543-4.png" width="2232" /></a></p> 
<p>3. Enter your API endpoint URL. Test connection and click system Info.</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/04/09/imageComputeBlog-2543-5-2.png"><img alt="Testing the connection" class="alignnone size-full wp-image-26054" height="1206" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/04/09/imageComputeBlog-2543-5-2.png" width="2230" /></a></p> 
<h3><strong>Test the Application</strong></h3> 
<p><strong>a. Customer Analysis</strong> — Enter one or more User IDs to get more information on the customer behavior: engagement scores, conversion rates, purchase patterns, and AI-generated customer segments</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/04/09/imageComputeBlog-2543-6.png"><img alt="Running customer analysis" class="alignnone size-full wp-image-26055" height="798" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/04/09/imageComputeBlog-2543-6.png" width="1240" /></a></p> 
<p><strong>b. Semantic Search – </strong>Enter natural language queries like “list high value customers from USA” in the Semantic Search and verify the results. Note that the response is very fast as the analytics data and FastEmbed models are loaded into memory during init stage</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/04/09/imageComputeBlog-2543-7-1.png"><img alt="Running semantic search" class="alignnone size-full wp-image-26056" height="798" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/04/09/imageComputeBlog-2543-7-1.png" width="1240" /></a></p> 
<p><strong>c. Cohort Analysis</strong> — Enter the query data to get Real-time segmentation by device, country, age group with aggregated metrics</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/04/09/imageComputeBlog-2543-8-1.png"><img alt="Running cohort analysis" class="alignnone size-full wp-image-26057" height="833" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/04/09/imageComputeBlog-2543-8-1.png" width="1227" /></a></p> 
<h3><strong>Observability</strong></h3> 
<p>AWS Lambda Managed Instances automatically publishes metrics to Amazon CloudWatch, giving you visibility into function performance and capacity utilization. Monitor <strong>InitDuration</strong> to track dataset and model load time at startup, <strong>MaxMemoryUsed</strong> to confirm your data fits within configured memory, and <strong>ProvisionedConcurrencySpilloverInvocations</strong> to detect when AWS Lambda Managed Instances capacity is exhausted.</p> 
<p>Enable <a href="https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Lambda-Insights.html" rel="noopener noreferrer" target="_blank"><strong>AWS Lambda Insights</strong></a> for enhanced per-invocation metrics including CPU time and memory utilization over time. Use <strong>Amazon CloudWatch Log Insights</strong> to query INIT_START, INIT_END, and REPORT log entries for initialization and memory details per invocation.</p> 
<h2><a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/04/09/imageComputeBlog-2543-9-1.png"><img alt="AWS Lambda Insights" class="alignnone size-full wp-image-26058" height="735" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/04/09/imageComputeBlog-2543-9-1.png" width="1660" /></a></h2> 
<h2><strong>What Makes This Better with AWS Lambda Managed Instances</strong></h2> 
<p>Without AWS Lambda Managed Instances, building this same application would require one of these alternatives:</p> 
<ul> 
 <li><strong>Option A: EC2 with auto-scaling</strong> — Full control, full responsibility: patching, scaling policies, load balancing, and deployment pipelines — all on you.</li> 
 <li><strong>Option B: Redesign for standard Lambda</strong> — Swap in-memory data for an external database and replace the ML model with <a href="https://aws.amazon.com/sagemaker/" rel="noopener noreferrer" target="_blank">Amazon SageMaker</a> endpoint. More latency, more cost, more complexity.</li> 
</ul> 
<p>With AWS Lambda Managed Instances, you write a single AWS Lambda function, define a Capacity Provider, and deploy with SAM. AWS Lambda handles the Amazon EC2 instances, scaling, and lifecycle, giving you the memory you need with the operational simplicity you want. The in-memory approach eliminates network latency and disk I/O, delivering consistent sub-200ms response times for complex analytics.</p> 
<h2><strong>Cost Considerations </strong></h2> 
<p>AWS Lambda Managed Instances uses Amazon EC2-based pricing with a management fee. For predictable workloads, you can leverage Amazon EC2 Savings Plans or Reserved Instances to reduce costs significantly.</p> 
<p><strong>Example cost comparison</strong> (us-east-1, 32 GB memory, 1M invocations/month):</p> 
<ul> 
 <li><strong>AWS Lambda (standard):</strong> ~$267/month (on-demand pricing)</li> 
 <li><strong>AWS Lambda Managed Instances:</strong> ~$180/month (with 1-year Compute Savings Plan)</li> 
 <li><strong>Savings:</strong> 33% reduction</li> 
</ul> 
<p>The cost benefits increase with higher memory configurations and sustained workloads that can take advantage of Amazon EC2 pricing discounts.</p> 
<h2><strong>Best Practices</strong></h2> 
<p>Based on experience building this solution, here are key recommendations:</p> 
<ul> 
 <li><strong>Memory sizing:</strong> Start with your dataset size plus 50% overhead for processing. Monitor Amazon CloudWatch metrics to optimize.</li> 
 <li><strong>Initialization strategy:</strong> Load large datasets during the init phase to amortize the cost across multiple invocations.</li> 
 <li><strong>Concurrency configuration:</strong> Set PerExecutionEnvironmentMaxConcurrency based on your workload’s I/O characteristics. Higher values work well for I/O-bound analytics.</li> 
 <li><strong>Data format:</strong> Use columnar formats like Parquet for efficient memory usage and fast loading.</li> 
 <li><strong>Monitoring:</strong> Track initialization duration, memory utilization, and invocation latency in Amazon CloudWatch to identify optimization opportunities.</li> 
</ul> 
<h2><strong>Cleanup</strong></h2> 
<p>When you’re done exploring the solution, it’s good practice to remove all provisioned resources to avoid ongoing charges. For the full cleanup commands and exact steps, refer to the project’s README.md in GitHub repository.</p> 
<h2><strong>Conclusion</strong></h2> 
<p>AWS Lambda Managed Instances opens up a new class of serverless applications that support larger AWS Lambda layer packages and more memory. Memory-intensive workloads — in-memory analytics, ML inference, graph processing, scientific computing — can now run with the simplicity of AWS Lambda and the resources of Amazon EC2. The customer analytics example demonstrates how in-memory processing with AWS Lambda Managed Instances delivers performance improvements over traditional database queries while maintaining serverless benefits like automatic scaling and pay-per-use pricing.</p> 
<p><strong>Ready to get started?</strong> Explore the <a href="https://docs.aws.amazon.com/lambda/latest/dg/lambda-managed-instances.html" rel="noopener noreferrer" target="_blank">AWS Lambda Managed Instances documentation</a> and try building your own memory-intensive serverless application. You can find the complete code for <a href="https://github.com/aws-samples/sample-lambda-managed-instances-analytics">this example on GitHub</a>.</p>
