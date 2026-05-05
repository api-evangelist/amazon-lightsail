---
title: "Enhancing network observability with new AWS Outposts racks LAG metrics"
url: "https://aws.amazon.com/blogs/compute/enhancing-network-observability-with-new-aws-outposts-racks-lag-metrics/"
date: "Thu, 30 Apr 2026 19:14:52 +0000"
author: "Adam Duffield"
feed_url: "https://aws.amazon.com/blogs/compute/feed/"
---
<p>When you deploy <a href="https://aws.amazon.com/outposts/rack/" rel="noopener noreferrer" target="_blank">AWS Outposts racks</a>, you can run AWS infrastructure and services in on-premises locations. Maintaining seamless connectivity, both to the <a href="https://aws.amazon.com/about-aws/global-infrastructure/regions_az/" rel="noopener noreferrer" target="_blank">AWS Region</a> and your on-premises network, is fundamental to delivering consistent, uninterrupted service to your applications. Implementing an observability strategy that uses available network metrics is key to understanding the health of this connectivity.</p> 
<p>In <a href="https://aws.amazon.com/blogs/compute/improving-network-observability-with-new-aws-outposts-racks-network-metrics/" rel="noopener noreferrer" target="_blank">August 2025</a>, we launched two new <a href="https://aws.amazon.com/cloudwatch/" rel="noopener noreferrer" target="_blank">Amazon CloudWatch</a> metrics, <code>VifConnectionStatus</code> and <code>VifBgpSessionState</code>, that helped provide greater visibility into these Layer 3 networking constructs. However, insight into Layer 2 networking was still missing. AWS has released a new <a href="https://docs.aws.amazon.com/outposts/latest/userguide/outposts-cloudwatch-metrics.html#outposts-metrics" rel="noopener noreferrer" target="_blank">metric</a> <code>LagStatus</code>, that provides greater visibility into the hybrid infrastructure connectivity for both <a href="https://docs.aws.amazon.com/outposts/latest/userguide/index.html" rel="noopener noreferrer" target="_blank">first-generation</a> and <a href="https://docs.aws.amazon.com/outposts/latest/network-userguide/index.html" rel="noopener noreferrer" target="_blank">second-generation</a> Outpost racks.</p> 
<h2>Link aggregation group overview</h2> 
<p><a href="https://docs.aws.amazon.com/outposts/latest/userguide/local-rack.html#link-aggregation" rel="noopener noreferrer" target="_blank">Link aggregation</a> combines multiple physical Ethernet connections into one logical link, referred to as a link aggregation group (LAG). This consolidation delivers benefits such as increased aggregate bandwidth and built-in redundancy through fault-tolerant connections between network devices. AWS Outposts uses LAG connections between Outpost network devices (ONDs) and customer network devices (CNDs). The links from each Outpost network device are aggregated into an Ethernet LAG to represent a single network connection.</p> 
<div class="wp-caption alignnone" id="attachment_25960" style="width: 1218px;">
 <img alt="Figure : Second-Generation Outposts Rack network connections" class="size-full wp-image-25960" height="646" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/04/02/compute-2553-image-1.png" width="1208" />
 <p class="wp-caption-text" id="caption-attachment-25960">Figure : Second-Generation Outposts Rack network connections</p>
</div> 
<p>Each LAG between an Outpost network device and a customer local network device is configured as an IEEE 802.1q Ethernet trunk. This enables the use of multiple VLANs for network segmentation between data paths. Each Outpost has the following VLANs to communicate with local network devices:</p> 
<ul> 
 <li>Service link VLAN – Enables communication between the Outpost and customer network devices to establish a service link path to the AWS Region.</li> 
 <li>Local gateway VLAN(s) – (If exists, and as single or multiple LGW routing domains), enables communication between Outpost and the customer network devices to establish a local gateway path to connect your Outpost subnets to the local area network.</li> 
</ul> 
<div class="wp-caption alignnone" id="attachment_25961" style="width: 1220px;">
 <img alt="Figure : Second-Generation Outposts Rack VLAN layout" class="size-full wp-image-25961" height="579" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/04/02/compute-2553-image-2.png" width="1210" />
 <p class="wp-caption-text" id="caption-attachment-25961">Figure : Second-Generation Outposts Rack VLAN layout</p>
</div> 
<h2>Using the LagStatus metric</h2> 
<p>The new <code>LagStatus</code> metric in CloudWatch provides visibility into the operational status of LAG connections between Outposts networking devices and on-premises infrastructure. The metric reports a binary status (1 for the LAG being UP, 0 for the LAG being down) and includes the <code>OutpostId</code> and <code>LagId</code> as dimensions to quickly identify non-operational resources.</p> 
<p>You can view this metric on the CloudWatch console. As with all operational telemetry, access to these metrics should be appropriately restricted to authorized principals. The metric data points are published at 5-minute intervals, and like all CloudWatch metrics, there might be a time lag in the metric data being published. In the navigation pane, choose&nbsp;<strong>All metrics</strong>, followed by&nbsp;<strong>Outposts</strong>&nbsp;under the AWS namespaces section. The Outposts namespace can only be viewed by the Outposts owner account, unless CloudWatch&nbsp;<a href="https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Unified-Cross-Account.html" rel="noopener noreferrer" target="_blank">cross-account observability</a>&nbsp;is configured.</p> 
<div class="wp-caption alignnone" id="attachment_25962" style="width: 1219px;">
 <img alt="Figure : CloudWatch Metrics view of the LagStatus metric" class="size-full wp-image-25962" height="468" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/04/02/compute-2553-image-3.png" width="1209" />
 <p class="wp-caption-text" id="caption-attachment-25962">Figure : CloudWatch Metrics view of the LagStatus metric</p>
</div> 
<p>While the <code>LagStatus</code> metric alone provides insight into the Outposts network connectivity, combining it with <code>VifConnectionStatus</code> and <code>VifBgpSessionState</code> delivers more immediate, actionable insights that expedite troubleshooting. In addition, to improve the clarity of the existing metrics, the related <code>LagId</code> is added as a new Outposts metric dimension. By observing the values of all three metrics, you can narrow down the potential cause of any issues. The following table gives some possible connectivity issue scenarios and how they can be identified using these metrics:</p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <tbody> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>LagStatus</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>LGW BGP</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>ServiceLink BGP</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Potential issue</strong></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">UP</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">UP</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">UP</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Recommended state – all components working</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">UP</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">UP</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">DOWN</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">ServiceLink BGP issue – configuration issue</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">UP</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">DOWN</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">UP</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">LGW BGP issue – configuration issue</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">UP</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">DOWN</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">DOWN</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Both BGP sessions down – configuration issue</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">DOWN</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">DOWN</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">DOWN</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Lag configuration issue or Physical failure</td> 
  </tr> 
 </tbody> 
</table> 
<p>With these metrics, you can use <a href="https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarm-combining.html" rel="noopener noreferrer" target="_blank">CloudWatch Composite Alarms</a> to alert operational teams when any of the components aren’t running as expected.</p> 
<p>To create a composite alarm, alarms must first be defined for all three of the individual metrics. This can be done from the console, CLI, or AWS CloudFormation. Following the principle of least privilege, ensure that IAM permissions are restricted to the minimum actions required for CloudWatch alarm creation. For more information, see the <a href="https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/permissions-reference-cw.html" rel="noopener noreferrer" target="_blank">CloudWatch documentation</a>. If you prefer, you can configure these individual alarms without notification actions enabled to reduce potential notification noise. Each <a href="https://docs.aws.amazon.com/outposts/latest/network-userguide/vif-vif-groups.html" rel="noopener noreferrer" target="_blank">virtual interface (VIF)</a> has its own set of metrics, so you would need to configure alarms for all VIFs used with your Outpost. The number of total VIFs will vary depending on the Outpost generation that’s deployed because of the different networking architectures.</p> 
<p>First-generation Outposts racks use four VIFs per rack (two for Service Link, two for Local Gateway). Second-generation racks require a minimum of eight VIFs (four for Service Link, four for Local Gateway), because they support <a href="https://aws.amazon.com/blogs/compute/simplify-network-segmentation-for-aws-outposts-racks-with-multiple-local-gateway-routing-domains/" rel="noopener noreferrer" target="_blank">multiple local gateway routing domains</a>, each with its own VIFs.</p> 
<p>An example alarm configuration as seen in the console for a single VIF is shown in the following figure 4.</p> 
<div class="wp-caption alignnone" id="attachment_25963" style="width: 1220px;">
 <img alt="Figure : Individual CloudWatch alarms for VIF status" class="size-full wp-image-25963" height="395" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/04/02/compute-2553-image-4.png" width="1210" />
 <p class="wp-caption-text" id="caption-attachment-25963">Figure : Individual CloudWatch alarms for VIF status</p>
</div> 
<p>After these individual alarms are created, a composite alarm can be created that monitors for any of the component metrics going into an alarm status. In the following example, the AWS Command Line Interface (AWS CLI) is used to create the composite alarm called composite-alarm-lag1 and send a notification using an <a href="https://aws.amazon.com/sns/" rel="noopener noreferrer" target="_blank">Amazon Simple Notification Service (Amazon SNS)</a> topic called outpost-network-alarms. As this topic carries infrastructure health data, it’s recommended to encrypt it using an <a href="https://aws.amazon.com/kms/" rel="noopener noreferrer" target="_blank">AWS Key Management Service</a> key and restrict the subscription policy to authorized principals.</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">aws cloudwatch put-composite-alarm \
  --alarm-name "composite-alarm-lag1" \
  --alarm-rule "ALARM(VifBgpSessionState-lgw-vif-xxxxxxxxxxxx) OR ALARM(VifConnectionStatus-lgw-vif-xxxxxxxxxxxx) OR ALARM(VifBgpSessionState-sl-vif-xxxxxxxxxxxx) OR ALARM(VifConnectionStatus-sl-vif-xxxxxxxxxxxx) OR ALARM(LagStatus-op-lag-xxxxxxxxxxxx)" \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:outpost-network-alarms \
  --region us-east-1</code></pre> 
</div> 
<p>You can use this granular monitoring to quickly identify and troubleshoot connectivity issues, particularly in scenarios where LAG status is up but VIF BGP status is down.</p> 
<h2>Conclusion</h2> 
<p>This post provides details about the newly released <code>LagStatus</code> CloudWatch metric, and how this metric can be used with existing metrics such as <code>VifConnectionStatus</code> and <code>VifBgpSessionState</code> to build a comprehensive network connectivity observability solution. The <code>LagStatus</code> metric is now available in all commercial AWS Regions and the AWS GovCloud (US-East) and AWS GovCloud (US-West) Regions where Outposts racks are available, for both first-generation and second-generation racks at no additional cost.</p> 
<p>For more information about Outposts rack networking patterns, see the&nbsp;<a href="https://docs.aws.amazon.com/whitepapers/latest/aws-outposts-high-availability-design/networking.html" rel="noopener noreferrer" target="_blank">Networking</a>&nbsp;section of the Outposts High Availability Design and Architecture Considerations whitepaper.</p> 
<p>Reach out to your AWS account team, or fill out this&nbsp;<a href="https://pages.awscloud.com/GLOBAL_PM_LN_outposts-features_2020084_7010z000001Lpcl_01.LandingPage.html" rel="noopener noreferrer" target="_blank">form</a>&nbsp;to learn more about observability for Outposts.</p>
