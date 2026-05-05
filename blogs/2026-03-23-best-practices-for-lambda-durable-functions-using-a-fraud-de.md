---
title: "Best practices for Lambda durable functions using a fraud detection example"
url: "https://aws.amazon.com/blogs/compute/best-practices-for-lambda-durable-functions-using-a-fraud-detection-example/"
date: "Mon, 23 Mar 2026 22:04:39 +0000"
author: "Debasis Rath"
feed_url: "https://aws.amazon.com/blogs/compute/feed/"
---
<p><a href="https://docs.aws.amazon.com/lambda/latest/dg/durable-functions.html" rel="noopener noreferrer" target="_blank">AWS Lambda durable functions</a>&nbsp;extend the Lambda programming model to build fault-tolerant multi-step applications and AI workflows using familiar programming languages. They preserve progress despite interruptions and execution can suspend for up to one year, for human approvals, scheduled delays, or other external events, without incurring compute charges for on-demand functions.</p> 
<p>This post walks through a fraud detection system built with durable functions. It also highlights the best practices that you can apply to your own production workflows, from approval processes to data pipelines to AI agent orchestration. You will learn how to handle concurrent notifications, wait for customer responses, and recover from failures without losing progress. If you are new to durable functions, check out the <a href="https://aws.amazon.com/blogs/compute/building-fault-tolerant-long-running-application-with-aws-lambda-durable-functions/" rel="noopener noreferrer" target="_blank">Introduction to Durable Functions blog post</a>&nbsp;first.</p> 
<h2><strong>Fraud detection with human-in-the-loop</strong></h2> 
<p>Consider a credit card fraud detection system, which uses an AI agent to analyze incoming transactions and assign risk scores. For ambiguous cases (medium-risk scores), the system needs human approval before authorizing a transaction. The workflow branches based on risk:</p> 
<ul> 
 <li><strong>Low risk (score &lt; 3)</strong>: Authorize immediately</li> 
 <li><strong>High risk (score ≥ 5)</strong>: Send to the fraud department immediately</li> 
 <li><strong>Medium risk (score 3–4)</strong>: Suspend transaction, send SMS and email to cardholder, wait up to 24 hours for confirmation (wait time is customizable)</li> 
</ul> 
<div class="wp-caption alignnone" id="attachment_25907" style="width: 946px;">
 <img alt="Figure 1. Agentic Fraud Detection with durable Lambda functions" class="wp-image-25907 size-full" height="508" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/23/compute-2476-arch-diag.png" width="936" />
 <p class="wp-caption-text" id="caption-attachment-25907">Figure 1. Agentic Fraud Detection with durable Lambda functions</p>
</div> 
<p>With human-in-the-loop workflows, response times can vary from minutes to hours. These delays introduce the need to durably preserve the state without consuming compute resources while waiting. With financial systems, we must also implement idempotency to guard against duplicate messages (invocations) and recover from failures without reprocessing completed work. To address these requirements, developers implement polling patterns with external state stores like Amazon DynamoDB or Amazon Simple Storage Service (Amazon S3) to manage idempotency, pay for idle compute while waiting for callbacks, introduce external orchestration components, or build asynchronous message-driven systems to handle long-processing tasks.</p> 
<p>Lambda durable functions provide a new alternative to address these challenges through durable execution, a pattern that uses checkpoints (saved state snapshots) to preserve progress and replays from saved state to recover from failures or resume after waiting. With checkpointing capabilities, you no longer need to pay Lambda compute charges while waiting, whether for callbacks, scheduled delays, or external events. Learn how to implement durable functions using the complete fraud detection implementation at this&nbsp;<a href="https://github.com/aws-samples/sample-lambda-durable-functions/tree/main/Industry%20Solutions/Financial%20Services%20%28FSI%29/FraudDetection" rel="noopener noreferrer" target="_blank">GitHub repository</a>. You can deploy it to your AWS account and experiment with the code as you read. The repository includes deployment instructions, sample data, and helper functions for testing.</p> 
<p>As we walk through the code, we’ll focus on best practices for designing workflows with durable execution and how to apply these patterns correctly in production workflows.</p> 
<h2><strong>Design steps to be idempotent</strong></h2> 
<p>Durable execution is designed to preserve progress through checkpoints and replay, but that reliability model means step logic can execute more than once. When steps retry, how do you prevent duplicate actions like charges to the credit card or repeated customer SMS or email notifications?</p> 
<p>Durable functions use&nbsp;<strong><em>at-least-once execution</em></strong>&nbsp;by default, executing each step at least one time, potentially more if failures occur. When a step fails, it retries. There are two strategies to design idempotent steps that prevent duplicate side effects: using external API idempotency keys and using the at-most-once step semantics built into durable functions.</p> 
<p><strong>Strategy A</strong>: External API Idempotency Keys</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">// Strategy A: Use external API idempotency keys
await context.step(`authorize-${tx.id}`, async () =&gt; {
  return payment.charges.create({
    amount: tx.amount,
    currency: 'usd',
    idempotency_key: `tx-${tx.id}`, // Prevents duplicate charges
    description: `Transaction ${tx.id}`
  });
});</code></pre> 
</div> 
<p>Notice the configuration:</p> 
<ul> 
 <li><strong>idempotency_key in API call</strong>: If the step retries, the payment processor recognizes it’s a duplicate request and returns the original result</li> 
 <li><strong>Defense in depth</strong>: Two layers of protection: Lambda checkpointing and external API idempotency</li> 
</ul> 
<p>Each layer provides independent protection. If Lambda’s checkpoint fails, the external API prevents duplicate charges. For legacy systems without idempotency support, where it’s critical that an operation is not executed more than once, use at-most-once semantics:</p> 
<p><strong>Strategy B</strong>: Use At-Most-Once Semantics</p> 
<p>For legacy systems without idempotency support, use at-most-once execution, a delivery feature that executes each step zero or one time, never more:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">// Strategy B: At-most-once step semantics
await context.step("charge-legacy-system", async () =&gt; {
  return await legacyPaymentSystem.charge(tx.amount);
}, {
  semantics: StepSemantics.AtMostOncePerRetry,
  retryStrategy: createRetryStrategy({ maxAttempts: 0 })
});</code></pre> 
</div> 
<p>This checkpoints before step execution, preventing the step from re-execution on retries. The tradeoff? If the step fails, you must decide whether to retry (risking duplicates) or fail the entire workflow.</p> 
<p>Use idempotency for critical side effects like payment processing, database writes, external API calls, state transitions, and resource provisioning. Read more about idempotency&nbsp;<a href="https://docs.aws.amazon.com/lambda/latest/dg/durable-execution-idempotency.html" rel="noopener noreferrer" target="_blank">here</a>.</p> 
<h2><strong>Prevent duplicate executions with DurableExecutionName</strong></h2> 
<p>Idempotent steps prevent duplicate side effects within a single execution, but what about duplicate workflow executions running concurrently? For example, duplicate messages in the queue, users clicking “Submit” multiple times in the UI, or the same event arriving via multiple channels like webhook and API. Without protection, each invocation creates a separate durable execution, potentially running the fraud check multiple times, sending duplicate notifications, and creating confusion about which execution is authoritative. Durable functions provide <code>DurableExecutionName</code> to help ensure only one concurrent execution per unique name.</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">// Invoke fraud detection function with execution name
await lambda.invoke({
  FunctionName: 'fraud-detection',
  InvocationType: 'Event',
  DurableExecutionName: `tx-${transactionId}`,
  Payload: JSON.stringify({
    id: transactionId,
    amount: 6500,
    location: 'New York, NY',
    vendor: 'Amazon.com'
  })
});</code></pre> 
</div> 
<p>Notice the configuration:</p> 
<ul> 
 <li><strong>DurableExecutionName: tx-${transactionId}</strong>: Uses the transaction ID as a unique execution identifier</li> 
 <li><strong>InvocationType: ‘Event’</strong>: Asynchronous invocation supports long-running workflows beyond 15 minutes</li> 
 <li><strong>One execution per transaction</strong>: If three invocations arrive with the same transaction ID, only the first creates an execution. Subsequent requests with the same execution name and payload receive an idempotent response returning the existing execution’s ARN, rather than creating a new execution.</li> 
</ul> 
<p>Lambda durable functions work with Lambda event sources, including event source mappings (ESM) such as&nbsp;<a href="https://aws.amazon.com/sqs/" rel="noopener noreferrer" target="_blank">Amazon Simple Queue Service (Amazon SQS)</a>,&nbsp;<a href="https://aws.amazon.com/kinesis/" rel="noopener noreferrer" target="_blank">Amazon Kinesis</a>, and DynamoDB Streams. ESMs invoke durable functions synchronously and inherit Lambda’s&nbsp;<a href="https://docs.amazonaws.cn/en_us/lambda/latest/dg/durable-invoking-esm.html" rel="noopener noreferrer" target="_blank">15-minute invocation limit</a>. Therefore, like direct Request/Response invocations, durable functions executions using event source mappings cannot exceed 15 minutes.</p> 
<p>For workflows exceeding 15 minutes, use an intermediary Lambda function between the event source mapping and durable function:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">// Intermediary function for SQS -&gt; Durable function
export const handler = async (event) =&gt; {
  for (const record of event.Records) {
    const transaction = JSON.parse(record.body);
    await lambda.invoke({
      FunctionName: process.env.FRAUD_DETECTION_FUNCTION,
      InvocationType: 'Event',
      DurableExecutionName: `tx-${transaction.id}`,
      Payload: JSON.stringify(transaction)
    });
  }
};</code></pre> 
</div> 
<p>This removes the 15-minute limit, allows executions up to one year, and enables custom execution name parameters for idempotency. Use&nbsp;<a href="https://aws.amazon.com/powertools-for-aws-lambda/" rel="noopener noreferrer" target="_blank">Powertools for AWS Lambda</a> to prevent duplicate invocations of the durable function when the event source mapping retries the intermediary function. Additionally, configure failure handling for your event source to capture failed invocations for future redrive or replay. For example, dead letter queues for SQS, or on-failure destinations for other event sources.</p> 
<h2><strong>Match timeouts to invocation type</strong></h2> 
<p>One important configuration detail ties these patterns together: matching your timeout settings to your invocation type. Lambda synchronous invocations (<code>RequestResponse</code>) have a hard 15-minute timeout limit. If you configure a durable execution to run for 24 hours but invoke it synchronously, the synchronous invocation fails immediately with an exception. Durable functions support workflows up to one year when invoked asynchronously.</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">// Lambda function configuration
{
  FunctionName: 'fraud-detection',
  Timeout: 300,
  MemorySize: 512,
  DurableConfig: {
    ExecutionTimeout: 90000
  }
}</code></pre> 
</div> 
<p>And invoke asynchronously:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">// Async invocation for long-running workflow
await lambda.invoke({
  FunctionName: 'fraud-detection',
  InvocationType: 'Event',
  DurableExecutionName: `tx-${transactionId}`,
  Payload: JSON.stringify(transaction)
});</code></pre> 
</div> 
<p>Notice the configuration:</p> 
<ul> 
 <li><strong>Timeout: 300</strong>: Lambda function timeout (5 minutes in this example, up to a maximum of 15 minutes). This defines the maximum duration for each active execution phase, including the initial invocation and any subsequent replays. Set this to cover the longest expected active processing time in your workflow.</li> 
 <li><strong>ExecutionTimeout: { hours: 25 }</strong>: Durable execution timeout covers the workflow’s expected total duration including suspension periods. Set this slightly above the longest wait timeout to avoid edge cases.</li> 
 <li><strong>InvocationType: ‘Event’</strong>: Asynchronous invocation removes the 15-minute limit and enables executions up to one year.</li> 
</ul> 
<p>The Lambda function timeout applies to active execution phases (AI calls, notification sending). During suspension (waiting for callbacks), the function isn’t running, so this timeout doesn’t apply. Setting the durable execution timeout to a meaningful boundary prevents workflows from running longer than expected. Without an explicit timeout, executions can run up to the maximum lifetime of one year.</p> 
<table border="1px" cellpadding="10px" class="styled-table"> 
 <tbody> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;"></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Synchronous (RequestResponse)</strong></td> 
   <td style="padding: 10px; border: 1px solid #dddddd;"><strong>Asynchronous (Event)</strong></td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Total duration</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Under 15 minutes</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Up to 1 year</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Caller needs result</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Yes</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">No</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Idempotency support</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Yes</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Yes</td> 
  </tr> 
  <tr> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Waits with suspension</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Yes</td> 
   <td style="padding: 10px; border: 1px solid #dddddd;">Yes</td> 
  </tr> 
 </tbody> 
</table> 
<h2><strong>Execute Concurrent Operations with context.parallel()</strong></h2> 
<p>In the fraud detection workflow, the system notifies the cardholder through multiple channels such as SMS and email. Preserving business logic when executing parallel workflows introduces code complexities such as managing execution state across branches, handling synchronization, and coordinating branch completion. Durable functions simplify parallel workflow implementation using&nbsp;<code>context.parallel()</code>, which executes branches concurrently while maintaining durable checkpoints for each branch and provides configurable options to handle partial completions. By checkpointing and managing the state internally, durable functions help make sure that the state is preserved even if there are retries or failures. Note that&nbsp;<code>context.parallel()</code>&nbsp;manages the internal execution state for each branch. If your branches interact with a shared external state (such as a database), you’re responsible for managing concurrent access to that external state.</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">// Human-in-the-loop: verify via email AND SMS (first response wins)
let verified = await context.parallel("human-verification", [
  (ctx) =&gt; ctx.waitForCallback("SendVerificationEmail",
    async (callbackId) =&gt; sendCustomerNotification(callbackId, 'email', tx)
  ),
  (ctx) =&gt; ctx.waitForCallback("SendVerificationSMS",
    async (callbackId) =&gt; sendCustomerNotification(callbackId, 'sms', tx)
  )
], {
  maxConcurrency: 2,
  completionConfig: {
    minSuccessful: 1 // Continue after 1 success
  }
});</code></pre> 
</div> 
<p>Notice the configuration:</p> 
<ul> 
 <li><strong>maxConcurrency: 2</strong>: Both notifications sent at the same time</li> 
 <li><strong>minSuccessful: 1</strong>: We only need one channel to succeed, whichever responds first wins</li> 
</ul> 
<p>Each parallel branch waits for its callback independently, and the durable execution checkpoints each branch as part of the execution state. Using the&nbsp;<code>minSuccessful</code>&nbsp;parameter, you control the minimum number of successful branch executions required for the parallel operation to complete. In this example, only one of the two branches needs to succeed. Verifications through SMS or email are both valid, and the workflow resumes as soon as either channel completes successfully. We call this the&nbsp;<strong>first-response-wins</strong>&nbsp;pattern. This pattern works well when you only need a single successful result from any parallel branch and want the remaining branches to stop blocking progress.</p> 
<p>But what happens if neither channel responds? Without timeouts, this workflow could remain suspended for up to the configured execution lifetime.</p> 
<h2><strong>Always configure callback timeouts</strong></h2> 
<p>Let’s add timeout protection to the parallel verification from the previous section.&nbsp;<code>context.waitForCallback()</code>&nbsp;accepts a&nbsp;timeout&nbsp;option that bounds how long each branch waits before throwing an exception. By wrapping the parallel call in a try/catch, you can implement fallback logic when users don’t respond in time.</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">// Enhanced: parallel verification with timeout and error handling
let verified;
try {
  verified = await context.parallel("human-verification", [
    (ctx) =&gt; ctx.waitForCallback("SendVerificationEmail",
      async (callbackId) =&gt; sendCustomerNotification(callbackId, 'email', tx),
      { timeout: { days: 1 } }  // Wait up to 1 day for email response
    ),
    (ctx) =&gt; ctx.waitForCallback("SendVerificationSMS",
      async (callbackId) =&gt; sendCustomerNotification(callbackId, 'sms', tx),
      { timeout: { days: 1 } }  // Wait up to 1 day for SMS response
    )
  ], {
    maxConcurrency: 2,
    completionConfig: {
      minSuccessful: 1
    }
  });
} catch (error) {
  const isTimeout = error.message?.includes("timeout");
  if (isTimeout) {
    context.logger.warn("Customer verification timeout", { error, txId: tx.id });
    // Fallback: escalate to fraud department
    return await context.step("sendToFraudDepartment", async () =&gt;
      sendToFraudDepartment(tx, true)
    );
  }
  throw error; // Re-throw non-timeout errors
}</code></pre> 
</div> 
<p>Notice what changed from the previous section:</p> 
<ul> 
 <li><strong>timeout: { days: 1 }</strong>: Each callback branch now has a maximum wait time of 1 day. If neither the email nor SMS callback arrives within that window, a timeout exception is thrown.</li> 
 <li><strong>try/catch with timeout detection</strong>: The catch block distinguishes between timeout errors and other exceptions. When a timeout occurs, the workflow implements fallback logic by escalating the transaction to the fraud department, while non-timeout errors are re-thrown to be handled by the durable execution retry mechanism.</li> 
</ul> 
<p>Without this error handling, the entire execution fails unhandled. The timeout also works with the&nbsp;<code>minSuccessful</code>&nbsp;configuration: if one branch times out but the other succeeds, the parallel operation still completes successfully since only one successful result is required.</p> 
<p>For advanced use cases where the callback handler performs long-running work, you can also configure a&nbsp;<code>heartbeatTimeout</code>&nbsp;to detect stalled callbacks before the main timeout expires. See the&nbsp;<a href="https://docs.aws.amazon.com/lambda/latest/dg/durable-functions.html" rel="noopener noreferrer" target="_blank">Lambda Developer Guide</a>&nbsp;for details.</p> 
<p>Use callback timeouts for human approvals, external API callbacks, asynchronous processing, and third-party integrations.</p> 
<h2><strong>Putting it all together: complete fraud detection implementation</strong></h2> 
<p>Now let’s see how all the best practices work together in the complete fraud detection workflow:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">import { withDurableExecution } from "@aws/durable-execution-sdk-js";
import { BedrockAgentCoreClient, InvokeAgentRuntimeCommand } from "@aws-sdk/client-bedrock-agentcore";

const agentRuntimeArn = process.env.AGENT_RUNTIME_ARN;
const agentRegion = process.env.AGENT_REGION || 'us-east-1';
const client = new BedrockAgentCoreClient({ region: agentRegion });

export const handler = withDurableExecution(async (event, context) =&gt; {
  const tx = {
    id: event.id,
    amount: event.amount,
    location: event.location,
    vendor: event.vendor
  };

  // AI fraud assessment with error handling
  tx.score = await context.step("fraudCheck", async () =&gt; {
    try {
      const payloadJson = JSON.stringify({ input: { amount: tx.amount } });
      const command = new InvokeAgentRuntimeCommand({
        agentRuntimeArn: agentRuntimeArn,
        qualifier: 'DEFAULT',
        payload: Buffer.from(payloadJson, 'utf-8'),
        contentType: 'application/json',
        accept: 'application/json'
      });
      const response = await client.send(command);
      const responseText = await response.response.transformToString();
      const result = JSON.parse(responseText);
      return result?.output?.risk_score ?? 5;  // Default to high-risk if score unavailable
    } catch (error) {
      context.logger.error("Fraud check failed", { error, txId: tx.id });
      return 5;
    }
  });

  // Route based on AI decision
  if (tx.score &lt; 3) {
    // Best Practice: Idempotent authorization
    return await context.step(`authorize-${tx.id}`, async () =&gt;
    authorizeTransaction(tx, { idempotency_key: `tx-${tx.id}` })
    );
  }

  if (tx.score &gt;= 5) {
    return await context.step(`sendToFraudDepartment-${tx.id}`, async () =&gt;
      sendToFraudDepartment(tx)
    );
  }

  // Medium risk: need human verification
  await context.step(`suspend-${tx.id}`, async () =&gt; suspendTransaction(tx));

  // Best Practice: Concurrent operations with timeout configuration
  let verified;
  try {
    verified = await context.parallel("human-verification", [
      (ctx) =&gt; ctx.waitForCallback("SendVerificationEmail",
        async (callbackId) =&gt; sendCustomerNotification(callbackId, 'email', tx),
        { timeout: { days: 1 } }
      ),
      (ctx) =&gt; ctx.waitForCallback("SendVerificationSMS",
        async (callbackId) =&gt; sendCustomerNotification(callbackId, 'sms', tx),
        { timeout: { days: 1 } }
      )
    ], {
      maxConcurrency: 2,
      completionConfig: {
        minSuccessful: 1
      }
    });
  } catch (error) {
    const isTimeout = error.message?.includes("timeout");
    context.logger.warn(
      isTimeout ? "Customer verification timeout" : "Customer verification failed",
      { error, txId: tx.id }
    );
    return await context.step(`timeout-escalate-${tx.id}`, async () =&gt;
      sendToFraudDepartment(tx, true)
    );
  }

  // Idempotent final step with idempotency key
  return await context.step(`finalize-${tx.id}`, async () =&gt; {
    const action = !verified.hasFailure &amp;&amp; verified.successCount &gt; 0
      ? "authorize"
      : "escalate";
    if (action === "authorize") {
      return authorizeTransaction(tx, true, { idempotency_key: `finalize-${tx.id}` });
    }
    return sendToFraudDepartment(tx, true);
  });
});</code></pre> 
</div> 
<p>Notice how the best practices work together:&nbsp;<code>context.parallel()</code>&nbsp;sends SMS and email concurrently, resuming when either channel responds. Both callbacks configure 1-day timeouts with try/catch handling that escalates on timeout. The&nbsp;<code>DurableExecutionName: tx-${transactionId}</code>&nbsp;parameter (specified at invocation time, shown in the following CLI example) provides execution-level deduplication, while idempotency keys in the authorization steps prevent duplicate charges at the application layer. Asynchronous invocation (<code>InvocationType: 'Event'</code>) enables the 24-hour wait period.</p> 
<p>Once deployed, invoke the function asynchronously with a sample transaction to see it in action:</p> 
<div class="hide-language"> 
 <pre><code class="lang-code">transactionId="123456789"
aws lambda invoke \
  --function-name "fraud-detection:$LATEST" \
  --invocation-type Event \
  --durable-execution-name "tx-${transactionId}" \
  --cli-binary-format raw-in-base64-out \
  --payload "{\"id\": \"${transactionId} \", \"amount\": 6500, \"location\": \"New York, NY\", \"vendor\": \"Amazon.com\"}" \
  --region us-east-2 \
  response.json</code></pre> 
</div> 
<p>Upon successful invocation, you can view the execution state in the Lambda console’s durable operations view. The execution shows a suspended state, waiting for customer response:</p> 
<div class="wp-caption alignnone" id="attachment_25859" style="width: 911px;">
 <img alt="Figure 2: Suspended execution state" class="size-full wp-image-25859" height="495" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/17/compute-2476-image-2.png" width="901" />
 <p class="wp-caption-text" id="caption-attachment-25859">Figure 2: Suspended execution state</p>
</div> 
<p>Notice the <code>fraudCheck</code> and <code>suspendTransaction</code> steps show as succeeded with checkpointed results. The human-verification parallel operation shows that both SMS and email branches started. The timeline shows the function in a suspended state. Simulate a customer response by sending a callback success through the console, AWS Command Line Interface (AWS CLI) or Lambda API:</p> 
<div class="hide-language"> 
 <div class="hide-language"> 
  <pre><code class="lang-code">aws lambda send-durable-execution-callback-success \
	--callback-id &lt;CALLBACK_ID_FROM_EMAIL_OR_SMS&gt; \
	--result '{"status":"approved","channel":"email"}' \
	--cli-binary-format raw-in-base64-out</code></pre> 
 </div> 
</div> 
<div class="wp-caption alignnone" id="attachment_25860" style="width: 911px;">
 <img alt="Figure 3: Completed execution with customer approval" class="size-full wp-image-25860" height="597" src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/03/17/compute-2476-image-3.png" width="901" />
 <p class="wp-caption-text" id="caption-attachment-25860">Figure 3: Completed execution with customer approval</p>
</div> 
<p>After receiving the customer’s approval, the durable execution resumes from its checkpoint, authorizes the transaction, and completes. The execution spanned hours but consumed only seconds of compute time.</p> 
<h2><strong>Conclusion</strong></h2> 
<p>With durable functions, Lambda extends beyond single-event processing to power core business processes and long-running workflows, while retaining the operational simplicity, reliability, and scale that define Lambda. You can build applications that run for days or months, survive failures, and resume where they left off, all within the familiar event-driven programming model.</p> 
<p>Deploy the fraud detection workflow from our&nbsp;<a href="https://github.com/aws-samples/sample-lambda-durable-functions/tree/main" rel="noopener noreferrer" target="_blank">GitHub repository</a>&nbsp;and experiment with human-in-the-loop patterns in your own account. For core concepts, see&nbsp;<a href="https://aws.amazon.com/blogs/compute/building-fault-tolerant-long-running-application-with-aws-lambda-durable-functions/" rel="noopener noreferrer" target="_blank">Introduction to AWS Lambda Durable Functions</a>. For comprehensive documentation, see the&nbsp;<a href="https://docs.aws.amazon.com/lambda/latest/dg/durable-functions.html" rel="noopener noreferrer" target="_blank">Lambda Developer Guide</a>. Browse&nbsp;<a href="https://serverlessland.com/search?search=Durable+function" rel="noopener noreferrer" target="_blank">Serverless Land</a>&nbsp;for reference architectures and discover where durable execution fits in your designs.</p> 
<p>Share your feedback, questions, and use cases in the SDK repositories or on&nbsp;<a href="https://repost.aws/" rel="noopener noreferrer" target="_blank">re:Post</a>.</p>
