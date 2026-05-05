---
title: "AWS Outposts monitoring and reporting: A comprehensive Amazon EventBridge solution"
url: "https://aws.amazon.com/blogs/compute/aws-outposts-monitoring-and-reporting-a-comprehensive-amazon-eventbridge-solution/"
date: "Tue, 14 Apr 2026 16:18:12 +0000"
author: "Matt Price"
feed_url: "https://aws.amazon.com/blogs/compute/feed/"
---
<p>Organizations using <a href="https://aws.amazon.com/outposts/rack/" rel="noopener noreferrer" target="_blank">AWS Outposts racks</a> commonly manage capacity from a single AWS account and share resources through <a href="https://aws.amazon.com/ram/" rel="noopener noreferrer" target="_blank">AWS Resource Access Manager</a> (AWS RAM) with other AWS accounts (consumer accounts) within <a href="https://aws.amazon.com/organizations/" rel="noopener noreferrer" target="_blank">AWS Organizations</a>. In this post, we demonstrate one approach to create a multi-account serverless solution to surface costs in shared AWS Outposts environments using <a href="https://aws.amazon.com/eventbridge/" rel="noopener noreferrer" target="_blank">Amazon EventBridge</a>, <a href="https://aws.amazon.com/lambda/" rel="noopener noreferrer" target="_blank">AWS Lambda</a>, and <a href="https://aws.amazon.com/dynamodb/" rel="noopener noreferrer" target="_blank">Amazon DynamoDB</a>. This solution reports on instance runtime and allocated storage for <a href="https://aws.amazon.com/ec2/" rel="noopener noreferrer" target="_blank">Amazon Elastic Compute Cloud (Amazon EC2)</a>, <a href="https://aws.amazon.com/rds" rel="noopener noreferrer" target="_blank">Amazon Relational Database Services (Amazon RDS)</a>, and <a href="https://aws.amazon.com/ebs/" rel="noopener noreferrer" target="_blank">Amazon Elastic Block Store (Amazon EBS)</a> services running on Outposts racks. In turn, teams can track the cost of infrastructure associated with their workloads across AWS accounts. This solution is a framework that can be customized to meet your organization’s specific business objectives.</p> 
<h2>Solution overview</h2> 
<p>The following is the <a href="https://developer.hashicorp.com/terraform" rel="noopener noreferrer" target="_blank">Terraform</a>-based reference architecture used to represent the solution, including EventBridge, DynamoDB, and Lambda across a multi-account environment. Relevant launch events are tracked in EventBridge that invoke Lambda functions, which are logged in DynamoDB tables (<a href="https://github.com/aws-samples/sample-outposts-monitoring-and-reports" rel="noopener noreferrer" target="_blank">see sample code</a>). This allows reporting on captured event data through the <a href="https://boto3.amazonaws.com/v1/documentation/api/latest/index.html" rel="noopener noreferrer" target="_blank">AWS SDK for Python (Boto3)</a>.&nbsp;<a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/04/03/computeblog-2496-1.png"><img alt="AWS architecture diagram showing data collection and workload account integration with EventBridge, CloudTrail, and Outposts" class="alignnone size-full wp-image-25970" height="720" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/04/03/computeblog-2496-1.png" width="1280" /></a><br /> <em>Figure 1: Reference architecture for reporting solution on AWS Outposts&nbsp;</em></p> 
<h2>Prerequisites</h2> 
<p>The following prerequisites are necessary to implement this solution:</p> 
<ul> 
 <li>At least two active AWS accounts in the same <a href="https://aws.amazon.com/organizations/" rel="noopener noreferrer" target="_blank">AWS Organization</a> as the Outposts owner account. 
  <ul> 
   <li>One AWS account, which is the data collection account to store the event data (this doesn’t have to be the account that owns the Outposts).</li> 
   <li>Workload accounts where resources are deployed on Outposts.</li> 
  </ul> </li> 
 <li><a href="https://aws.amazon.com/cli/" rel="noopener noreferrer" target="_blank">AWS Command Line Interface (AWS CLI)</a> installed and configured on an administrative instance. For more information, see <a href="https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html" rel="noopener noreferrer" target="_blank">Installing, updating, and uninstalling the AWS CLI </a>in the AWS CLI documentation.</li> 
 <li>Terraform installed on the same administrative instance. For more information, see the <a href="https://learn.hashicorp.com/tutorials/terraform/install-cli" rel="noopener noreferrer" target="_blank">Terraform documentation</a>.</li> 
 <li>Make sure that you have the necessary <a href="https://aws.amazon.com/iam/" rel="noopener noreferrer" target="_blank">AWS Identity and Access Management (IAM)</a> permissions necessary to create the AWS resources using Terraform in all accounts.</li> 
 <li>Prior Experience with Terraform deployments on AWS Cloud. To increase your familiarity, you can explore <a href="https://learn.hashicorp.com/collections/terraform/aws-get-started" rel="noopener noreferrer" target="_blank">Get Started – AWS</a> on the HashiCorp website.</li> 
 <li>Access to clone the <a href="https://github.com/aws-samples/sample-outposts-monitoring-and-reports" rel="noopener noreferrer" target="_blank">AWS Outposts Monitoring and Reporting</a> git repository.</li> 
 <li>SDK for Python installed and configured on a local machine.</li> 
</ul> 
<h2>Walkthrough</h2> 
<p>The following sections walk you through how to deploy this solution.</p> 
<h3>Deploying in data collection account</h3> 
<p>Step 1: Create a bucket in-Region to hold the Terraform state file in the data collection account.</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">aws s3 mb s3://state-bucket-name</code></pre> 
</div> 
<p>Step 2:&nbsp;Clone the repository.On your local machine, clone the repository that contains the sample by running the following command:</p> 
<p>git clone <a href="https://github.com/aws-samples/sample-outposts-monitoring-and-reports.git" rel="noopener noreferrer" target="_blank">https://github.com/aws-samples/sample-outposts-monitoring-and-reports.git</a></p> 
<p>Navigate to the cloned repository by running the following command:cd sample-outposts-monitoring-and-reports/data_collection</p> 
<p>Step 3: Edit the providers.tf to configure the AWS provider.</p> 
<div class="hide-language"> 
 <pre><code class="lang-css">

provider "aws" {
&nbsp;&nbsp;region = ""
}</code></pre> 
</div> 
<p>Step 4: Edit the backend.tf to provide the Terraform state bucket and Outposts anchored <a href="https://aws.amazon.com/about-aws/global-infrastructure/regions_az/" rel="noopener noreferrer" target="_blank">AWS Region</a>.</p> 
<div class="hide-language"> 
 <pre><code class="lang-css">terraform {
&nbsp;&nbsp;backend "s3" {
&nbsp;&nbsp; &nbsp;bucket = ""
&nbsp;&nbsp; &nbsp;key &nbsp; &nbsp;= "terraform.tfstate"
&nbsp;&nbsp; &nbsp;region = ""
&nbsp;&nbsp;}
}</code></pre> 
</div> 
<p>Step 5: Modify the variables.tf.From the root directory of the cloned repository, modify the variables.tf file with the target Region and workload accounts as shown in the following example. The target Region is the collection destination.</p> 
<div class="hide-language"> 
 <pre><code class="lang-typescript">variable "aws_region" {
&nbsp;&nbsp;description = "AWS region for resources"
&nbsp;&nbsp;type &nbsp; &nbsp; &nbsp; &nbsp;= string
&nbsp;&nbsp;default &nbsp; &nbsp; = ""
}

variable "allowed_account_id" {
&nbsp;&nbsp;description = "AWS account ID allowed to put events to the event bus"
&nbsp;&nbsp;

}</code></pre> 
</div> 
<p>Initialize the configuration directory of the data collection account to download and install the providers defined in the configuration by running the following command:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">terraform init</code></pre> 
</div> 
<p>All resources are deployed with minimal permissions to serve as an example. We recommend viewing all configurations to make sure that they meet your organizational security policies.&nbsp;Step 6: Deploy infrastructure in the data collection account.Run terraform plan on the configuration to and review which resources are created:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">terraform plan</code></pre> 
</div> 
<p>When you have reviewed the plan, run the following command and enter “yes” to accept the changes and deploy:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">terraform apply</code></pre> 
</div> 
<p>Deployment should take less than 5 minutes. If you receive any errors, review the previously mentioned steps to ensure that you followed them in their entirety. If the errors persist, reach out to AWS Support for additional guidance.</p> 
<h3>Deploying in workload account</h3> 
<p>The data collection account receives events from EventBridge and performs intelligent analysis and storage from the AWS Outposts resource data.Step 1: Navigate to the workload account directory by running the following command:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">cd ../workload_account</code></pre> 
</div> 
<p>Step 2: Edit variables.tf to set up the Region and event bus <a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/reference-arns.html" rel="noopener noreferrer" target="_blank">Amazon Resource Name (ARN).&nbsp;</a></p> 
<div class="hide-language"> 
 <pre><code class="lang-typescript">variable "aws_region" {
&nbsp;&nbsp;description = "AWS region for resources"
&nbsp;&nbsp;type &nbsp; &nbsp; &nbsp; &nbsp;= string
&nbsp;&nbsp;default &nbsp; &nbsp; = ""
}

variable "event_bus_arn" {
&nbsp;&nbsp;description = "target event bus arn"
&nbsp;&nbsp;type &nbsp; &nbsp; &nbsp; &nbsp;= string
&nbsp;&nbsp;default &nbsp; &nbsp; = ""
}</code></pre> 
</div> 
<p>Edit the code to update the event bus name.</p> 
<p>Step 3: Run the following command to create the backend.tf and create the Terraform state bucket for each workload account.</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">./init-backend.sh</code></pre> 
</div> 
<p>This is an idempotent operation that creates a file from the template and a bucket with a fixed name including the account ID if it doesn’t exist.&nbsp;</p> 
<p>Step 4:&nbsp;Initialize the configuration directory of the Data Collection Account to download and install the providers defined in the configuration by running the following command:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">terraform init</code></pre> 
</div> 
<p>Step 5: Deploy the infrastructure in the Data Collection Account.Run a terraform plan on the configuration and review which resources are created:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">terraform plan</code></pre> 
</div> 
<p>After you have reviewed the plan, run the following command and enter “yes” to accept the changes and deploy:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">terraform apply</code></pre> 
</div> 
<p>Deployment should take less than 5 minutes. If you receive any errors, follow the troubleshooting steps in the previous section.</p> 
<p>At this point, any Amazon EC2 or Amazon RDS instances and Amazon EBS volumes are logged to the DynamoDB tables in the data collection account. Repeat Steps 3–5 for each workload account running resources on AWS Outposts with appropriate account credentials. If you’re deploying at scale and using <a href="https://docs.aws.amazon.com/controltower/latest/userguide/what-is-control-tower.html" rel="noopener noreferrer" target="_blank">AWS Control Tower</a> consider using <a href="https://docs.aws.amazon.com/controltower/latest/userguide/aft-overview.html" rel="noopener noreferrer" target="_blank">AWS Control Tower Account Factory for Terraform (AFT)</a>.</p> 
<h2>Running monthly reports</h2> 
<p>With this solution in place, reports can be generated on demand. These reports can be customized by modifying the Python example scripts shown to support your needs. Reports can be created from a local machine with credentials that have access to the DynamoDB tables in the data collection account. The examples were created from the source directory of the data collection account git repository.&nbsp;Run the following command to view the report for Amazon RDS usage in September 2025:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">./rds_runtime_calculator.py --year 2025 --month 9 --output rds_report.csv</code></pre> 
</div> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/04/03/computeblog-2496-2.png"><img alt="Spreadsheet showing RDS database instances with configuration details, storage allocation, and operational status in us-west-2 region" class="alignnone size-full wp-image-25971" height="155" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/04/03/computeblog-2496-2.png" width="1519" /></a></p> 
<p><em>Figure 2: Example of RDS runtime report&nbsp;</em></p> 
<p>&nbsp;</p> 
<p>Run the following command to view the report for Amazon EBS usage in September 2025:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">./ebs_volume_reporter.py --year 2025 --month 9 --output ebs_report.csv</code></pre> 
</div> 
<p>&nbsp;</p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/04/03/computeblog-2496-4.png"><img alt="EBS volume tracking table showing volume configurations, lifecycle hours, and active/deleted status in us-west-2" class="alignnone size-full wp-image-25973" height="95" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/04/03/computeblog-2496-4.png" width="1431" /></a></p> 
<p><em>Figure 3: Example of EBS usage report&nbsp;</em></p> 
<p>&nbsp;</p> 
<p>Run the following command to view the report for Amazon EC2 usage in September 2025:</p> 
<p><code>./ec2_runtime_calculator.py --month 9 --year 2025 --output ec2_report.csv</code></p> 
<p><a href="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/04/03/computeblog-2496-6.png"><img alt="EC2 instance tracking table showing c5.large instances with runtime hours and running/stopped status on AWS Outposts" class="alignnone size-full wp-image-25975" height="139" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/04/03/computeblog-2496-6.png" width="1431" /></a></p> 
<p><em>Figure 4: Example of EC2 runtime report&nbsp;</em></p> 
<p>&nbsp;</p> 
<h2>Cleaning up</h2> 
<p>Complete the following steps to clean up the resources that were deployed by this solution. For each workload account, complete the following:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">cd sample-outposts-monitoring-and-reports/workload_account</code></pre> 
</div> 
<div class="hide-language"> 
 <pre><code class="lang-code">terraform destroy </code></pre> 
</div> 
<p>Enter “yes” to proceed. You can then manually empty and remove the terraform state S3 bucket for that account.</p> 
<p>For the data collection, complete the following:</p> 
<div class="hide-language"> 
 <pre><code class="lang-js">cd ../data_collection</code></pre> 
</div> 
<div class="hide-language"> 
 <pre><code class="lang-js">terraform destroy</code></pre> 
</div> 
<p>Enter “yes” to proceed. You can then manually empty and remove the terraform state S3 bucket for that account.</p> 
<h2>Conclusion</h2> 
<p>Customers who have shared multi-account Outposts deployments can use this solution to create account level reporting for Outposts resources using real-time event capture and processing, state analysis and categorization, historical usage metrics, and serverless architecture.&nbsp;Teams can use this to visualize and report on the costs of running their workloads on Outposts. The event-driven design supports accurate tracking while maintaining low operational overhead. The solution scales effectively across multiple Outposts and accounts, providing a unified view of hybrid infrastructure. Keep in mind that you can extend the functionality described here to meet your business objectives.</p> 
<p>Deploy this solution today using the <a href="https://github.com/aws-samples/sample-outposts-monitoring-and-reports" rel="noopener noreferrer" target="_blank">GitHub repository</a> to gain financial insights to share with the tenants of your Outposts workload accounts.&nbsp;Reach out to your AWS account team, or fill out <a href="https://pages.awscloud.com/GLOBAL_PM_LN_outposts-features_2020084_7010z000001Lpcl_01.LandingPage.html" rel="noopener noreferrer" target="_blank">this form</a> to learn more about Outposts.</p>
