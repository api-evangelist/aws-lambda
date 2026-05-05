---
title: "Building fault-tolerant applications with AWS Lambda durable functions"
url: "https://aws.amazon.com/blogs/compute/building-fault-tolerant-long-running-application-with-aws-lambda-durable-functions/"
date: "Fri, 06 Feb 2026 16:54:39 +0000"
author: "Rahul Pisal"
feed_url: "https://aws.amazon.com/blogs/compute/category/compute/aws-lambda/feed/"
---
<p>Business applications often coordinate multiple steps that need to run reliably or wait for extended periods, such as customer onboarding, payment processing, or orchestrating large language model inference. These critical processes require completion despite temporary disruptions or system failures. Developers currently spend significant time implementing mechanisms to track progress, handle failures, and manage resources when waiting for external events, shifting focus from business logic to undifferentiated tasks.</p> 
<p>At re:Invent 2025,&nbsp;<a href="https://aws.amazon.com/" rel="noopener noreferrer" target="_blank">Amazon Web Services (AWS)</a>&nbsp;launched&nbsp;<a href="https://aws.amazon.com/lambda/" rel="noopener noreferrer" target="_blank">AWS Lambda</a>&nbsp;durable functions, a new capability extending Lambda’s event-driven programming model with built-in capabilities to build fault-tolerant multi-step applications and AI workflows using familiar programming languages. At its core, durable functions are regular Lambda functions, so your development and operational processes for Lambda continue to apply. However, when you create a Lambda function you can now enable durable execution, so that you can checkpoint progress, automatically recover from failures, and suspend execution for up to one year when waiting on long-running tasks, such as human-in-the-loop processes.</p> 
<h2>How Lambda durable functions work</h2> 
<p>When working with standard Lambda functions, your code runs from start to finish in a single invocation. If a failure occurs at any point during the execution, the entire function must be retried by the invoking event source. Any state that needs to be preserved between executions must be explicitly saved and retrieved. This is typically done by using external storage services such as&nbsp;<a href="https://aws.amazon.com/dynamodb/" rel="noopener noreferrer" target="_blank">Amazon DynamoDB</a>&nbsp;or&nbsp;<a href="https://aws.amazon.com/s3/" rel="noopener noreferrer" target="_blank">Amazon Simple Storage Service (Amazon S3</a>). Furthermore, you must typically guard against duplicate (concurrent) invocations of the same event and have a strategy to safely deploy updates while continuing to process events.</p> 
<p>In contrast, with Lambda durable functions, developers use durable operations such as “Steps” and “Waits” in the event handler to checkpoint progress, handle failures, and suspend execution during wait periods without incurring compute charges for on-demand functions. These durable operations and any optional state returned from them are automatically persisted by Lambda in a fully-managed durable execution backend. If failures occur during the execution, or if your function resumes its execution after being paused, Lambda invokes your function again, restoring (replaying) the previous state by executing the event handler from the start, but skipping over completed durable operations. To streamline this checkpoint/replay mechanism for developers, you can use the Lambda durable execution SDK to wrap or annotate your event handler, which enhances the existing Lambda context with several new methods like&nbsp;context.step()&nbsp;and context.wait(). Furthermore, you can use methods such as&nbsp;context.waitForCallback()&nbsp;to wait on external jobs or asynchronous processes, such as “human-in-the-loop” scenarios. The execution is paused until a&nbsp;SendDurableExecutionCallbackSuccess&nbsp;or&nbsp;SendDurableExecutionCallbackFailure&nbsp;response is sent to the Lambda API.</p> 
<h2>Getting started</h2> 
<p>Use the&nbsp;<a href="https://aws.amazon.com/serverless/sam/" rel="noopener noreferrer" target="_blank">AWS Serverless Application Model (AWS SAM)</a>&nbsp;to create a new durable function with&nbsp;<code>sam init</code>&nbsp;with an AWS Quick Start Template. Lambda durable functions are also supported by the&nbsp;<a href="https://aws.amazon.com/cdk/" rel="noopener noreferrer" target="_blank">AWS Cloud Development Kit (AWS CDK)</a>,&nbsp;<a href="https://aws.amazon.com/cli/" rel="noopener noreferrer" target="_blank">AWS Command Line Interface (AWS CLI),</a>&nbsp;<a href="https://aws.amazon.com/cloudformation/" rel="noopener noreferrer" target="_blank">AWS CloudFormation</a>&nbsp;and other infrastructure as code (IaC) frameworks such as Terraform.</p> 
<p>Consider the following function, which performs user onboarding. First, it creates a user profile based on some data, then it sends out an email for verification and waits until the user either confirms the email address, or a 24-hour timeout is reached. Finally, it sends out a confirmation.</p> 
<div class="hide-language"> 
 <pre><code class="lang-javascript">import {
  DurableContext,
  withDurableExecution,
} from '@aws/durable-execution-sdk-js';
export const handler = withDurableExecution(
  async (event: OnboardingEvent, context: DurableContext) =&gt; {
    try {    
      // Create user profile
      const profile = await context.step("create-profile", async () =&gt;
        createUserProfile(event.email, event.name)
      );
      // Wait for email verification via callback
      const verification = await context.waitForCallback(
        "wait-for-email-verification",
        async (callbackId) =&gt; {
          // Send email to user and pass callbackId
          await sendVerificationEmail(profile, callbackId);
        },
        {
          timeout: { hours: 24 } 
        }
      );
      // Send confirmation and welcome email
      const result = await context.step("complete-onboarding", async () =&gt; {
        if (!verification || !verification.verified) 
     return { ...profile, status: 'failed' };
        await sendWelcomeEmail(profile.email, profile.name);
        return { ...profile, status: 'active' };
      });
      return result;
    } catch (error) {
      // omitted 
    }
  }
);</code></pre> 
</div> 
<p>Durable functions have built-in and fully customizable error handling for steps. For example, if the profile was successfully created and verified, but a temporary error occurred when sending out the confirmation, then the step is retried. The retry skips over any previously completed checkpoints, such as the profile creation and callback. Only the code within the send confirmation step is run again.</p> 
<p>Next, you update the AWS SAM template to include your durable function. You create a Lambda durable function by including the DurableConfig setting for your function. Note that you currently cannot add a durable configuration to a function that was originally created without it. The&nbsp;ExecutionTimeout&nbsp;defines after which time the durable execution times out to protect against runaway or deadlock application bugs. This setting is separate from the invocation timeout, which defines for how long a single invocation can run. The maximum invocation timeout for a single function invocations remains unchanged at 15 minutes. With Lambda durable functions, you will typically see multiple invocations per durable execution, such as when using the wait capabilities in the SDK or automatic retries. You can set the ExecutionTimeout for up to one year when using asynchronous invocations.</p> 
<p>The&nbsp;RetentionPeriodInDays&nbsp;defines how long the execution data of a durable execution is available to you after executions complete.</p> 
<div class="hide-language"> 
 <pre><code class="lang-javascript">AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
 
Resources:
  UserOnboardingFunction:
    Type: AWS::Serverless::Function
    Properties:
      FunctionName: UserOnboardingFunction
      CodeUri: ./src
      Handler: index.handler
      Runtime: nodejs24.x
      Architectures:
        - x86_64
      MemorySize: 256
      Timeout: 60		   // Timeout for an individual invocation
      DurableConfig:		   // This makes the function a durable function
        ExecutionTimeout: 90000 // 25h timeout for the durable execution overall
        RetentionPeriodInDays: 7 
UserOnboardingFunctionRole:
    Type: AWS::IAM::Role
    // omitted for brevity</code></pre> 
</div> 
<p>You must include the necessary permissions for your function. For example, the&nbsp;<code>AWSLambdaBasicDurableExecutionRole</code> managed policy only allows the minimal&nbsp;<a href="https://aws.amazon.com/iam/" rel="noopener noreferrer" target="_blank">AWS Identity and Access Management (IAM)</a>&nbsp;actions to create/retrieve checkpoints and logs to increase security. Therefore, it does not include permissions to invoke other (durable) functions or manage callbacks. Refer to the&nbsp;<a href="https://docs.aws.amazon.com/lambda/latest/dg/durable-functions.html" rel="noopener noreferrer" target="_blank">documentation</a>&nbsp;for more details.</p> 
<h2>Testing locally</h2> 
<p>Before deploying your function, you can test it locally using AWS SAM local invoke.</p> 
<p><img src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/02/05/compute-2471-image-1.png" /></p> 
<p>AWS SAM locally invokes your function and runs the event handler until it reaches the&nbsp;<code>context.waitForCallback()</code>. To complete callbacks, AWS SAM offers new commands to interact with your durable functions. In this example, you send a&nbsp;<code>Success</code>&nbsp;response to complete the callback. You can also include relevant data in the response. You can send the response directly using the on-screen guide or using another AWS SAM CLI command from another process.</p> 
<p><code>sam local callback succeed &lt;your-callback-id&gt; --result '&lt;your data&gt;'</code></p> 
<p><img src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/02/05/compute-2471-image-2.png" /></p> 
<p>To inspect an execution, you can use AWS SAM to retrieve the durable execution history of your function, which includes details about steps, callbacks, and wait durations, as shown in the following example code.</p> 
<p><code>sam local execution history &lt;execution-arn&gt;</code></p> 
<p><img src="https://d2908q01vomqb2.cloudfront.net/1b6453892473a467d07372d45eb05abc2031647a/2026/02/05/compute-2471-image-3.png" /></p> 
<p>Depending on your use case, you can instead send a Failure response to a callback and handle those errors in your code. For example, by performing compensation logic in a subsequent step:</p> 
<p><code>sam local callback fail &lt;your-callback-id&gt; --error-data '&lt;your data&gt;'</code></p> 
<p>Now that you have verified that your function works as intended, deploy it to AWS using&nbsp;<code>sam&nbsp;deploy</code> command.</p> 
<h2>Best practices and considerations</h2> 
<p>Invoking a Lambda durable function requires a qualified&nbsp;<a href="https://docs.aws.amazon.com/IAM/latest/UserGuide/reference-arns.html" rel="noopener noreferrer" target="_blank">Amazon Resource Name (ARN),</a>&nbsp;such as an alias or version. We recommend that you don’t use the&nbsp;<code>$LATEST</code>&nbsp;qualifier except for rapid prototyping or local testing. Using explicit versions ensures that replays always happen with the same code with which the execution was started. This is to ensure deterministic execution and prevent inconsistencies when updating your function code during executions.</p> 
<p>We recommend bundling the durable execution SDK with your function code using your preferred package manager. The SDKs are fast-moving, so you can update dependencies as new features become available.</p> 
<p>There are&nbsp;<a href="https://docs.aws.amazon.com/lambda/latest/dg/durable-execution-sdk.html#durable-sdk-operations" rel="noopener noreferrer" target="_blank">other durable operations</a>&nbsp;in the Lambda durable functions SDK that you can use to build your application:</p> 
<ul> 
 <li><code>waitForCondition()</code>: Pauses the execution of your function until a condition is met. For example, the status of a job polled with an API. For this to work, you provide the waitStrategy and a check function to poll the status.</li> 
 <li><code>parallel()</code>: Runs multiple durable operations in parallel within the same function, with configurable options such as the maximum number of concurrent branches and desired failure behavior. This streamlines managing durability and checkpointing for simultaneous asynchronous actions.</li> 
 <li><code>map()</code>: Creates a durable operation and checkpoint for each item of an array, based on the provided mapping function. The items are processed concurrently.</li> 
 <li><code>invoke()</code>: Invokes another Lambda function and waits for its result. The SDK creates a checkpoint, invokes the target function, and resumes your function when the invocation completes. This enables function composition and workflow decomposition.</li> 
</ul> 
<p>Refer to the&nbsp;<a href="https://docs.aws.amazon.com/lambda/latest/dg/durable-execution-sdk.html" rel="noopener noreferrer" target="_blank">developer guide</a>&nbsp;for more details.</p> 
<p>Lambda compute charges apply to all invocations, including any replays. When using wait operations, the function suspends execution and, for on-demand functions, doesn’t incur duration charges until execution resumes. You’re also charged for durable operations, data written, and data retention. To learn more about Lambda durable functions pricing, refer to the&nbsp;<a href="https://aws.amazon.com/lambda/pricing/?trk=c4ea046f-18ad-4d23-a1ac-cdd1267f942c&amp;sc_channel=el" rel="noopener noreferrer" target="_blank">Lambda pricing</a>&nbsp;page.</p> 
<p>For the latest Region availability, visit the&nbsp;<a href="https://builder.aws.com/build/capabilities" rel="noopener noreferrer" target="_blank">AWS Capabilities by Region page</a>.</p> 
<h2>Conclusion</h2> 
<p>AWS Lambda durable functions extends the Lambda programming model to streamline building fault-tolerant, long-running applications using familiar programming patterns. You can use Lambda durable functions to write multi-step workflows in your preferred programming language, using built-in methods that automatically handle progress checkpointing and error recovery. This streamlines your architectures so that you can focus on your business logic, and optimize cost by charging only for active compute time.</p> 
<p>You can build durable functions for Python or Node.js based Lambda functions using the Lambda API,&nbsp;<a href="https://aws.amazon.com/console/" rel="noopener noreferrer" target="_blank">AWS Management Console</a>, AWS CLI, AWS CloudFormation, AWS SAM, AWS SDK, and AWS CDK.</p> 
<p>To get started, visit the&nbsp;<a href="https://docs.aws.amazon.com/lambda/latest/dg/durable-functions.html" rel="noopener noreferrer" target="_blank">Lambda Developer Guide</a>&nbsp;or watch the&nbsp;<a href="https://www.youtube.com/watch?v=XJ80NBOwsow" rel="noopener noreferrer" target="_blank">re:Invent breakout session</a>.</p>
