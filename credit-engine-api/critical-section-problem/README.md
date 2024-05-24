# Critical Section Problem in Credit Engine API
## What Critical Section Problem was encountered in implementing the Credit Engine?

The credit engine lambda takes approximately two minutes to complete a run. This means if Salesforce sends outbound messages directly to the credit engine lambda, it will not receive an acknowledgement within the standard timeout of 60 seconds (this timeout period is not adjustable in Salesforce). As a result, it will continue to send outbound messages, which will cause the credit engine to run several times. Not only is this inefficient and costly, in that each lambda invocation utilizes compute resources that have a cost associated with them, but —more importantly— multiple invocations can lead to multiple credit pulls on a single applicant if not handled properly. Multiple credit pulls, of course, have a direct and negative effect on a person’s credit score.

This issue is, in nature, a Critical Section Problem. A ‘critical section’ is a segment of code where shared data or resources may be accessed by multiple threads or processes concurrently. Examples of critical sections include code that updates a shared variable value or manages data transactions in a database. The risk presented by critical sections is the potential for multiple processes to be operating on a given resource concurrently, causing data inconsistencies and corruption.

In the case of the credit engine lambda and Salesforce outbound messages, because the outbound message retries can result in multiple invocations of the credit-engine lambda API for the same request, a single credit pull attempt can result in multiple pulls if a synchronization mechanism (code which facilitates mutually-exclusive access and activity to the critical section) is not put in place.

## Below are the steps that were taken to address our critical section issue:

### Step 1: Publisher-Subscriber Design

[*TO-DO: Explain how this works at a high level; whether it’s a popular approach and why.*]

In order to provide an acknowledgement to the Salesforce outbound message quickly enough to prevent retries, I implemented a publisher-subscriber (pub-sub) messaging pattern. In this setup, the Salesforce outbound message is routed to the ‘publisher’ API, which simply receives the event data, ‘publishes’ it to a message broker, and returns an acknowledgement to Salesforce. By passing the outbound message to the publisher api rather than the credit-engine itself the outbound message endpoint is able to return an acknowledgement to Salesforce quickly, mitigating the need for retries.

The message broker implemented here is an AWS SQS Queue — Amazon’s message queueing service. AWS describes message queues in the following terms:

> *A message queue is a form of asynchronous service-to-service communication used in serverless and microservices architectures. Messages are stored on the queue until they are processed and deleted. Each message is processed only once, by a single consumer.*
> 

 *-[AWS docs](https://aws.amazon.com/message-queue/)*

Once our publisher sends the Salesforce message to the SQS queue, it remains there until it has been processed and deleted by a subscribing service. The subscribing service here is, of course, the credit-engine lambda, which is configured as a ‘lambda-trigger’ on the queue. 

Lambda triggers, aptly named, are lambdas you can designate to be triggered when a new message enters the queue. Important SQS features that dictate how messages are processed once they enter the queue include dead letter queues, FIFO queues, and visibility timeouts. While it’s important to understand each of these in order to configure SQS appropriately for your needs, this section will focus on the visibility timeouts setting.

In SQS, the visibility timeout determines when messages can and can’t be accessed by other services. When a message enters the queue and is first consumed by a subscriber (here, the credit-engine lambda trigger), the visibility timeout period begins. During this period —measured in seconds, and which is defined when setting up the queue— the message is hidden in the queue, making it unavailable for other resources or threads to consume. This gives the initial resource that consumed the message time to process and delete it from the queue.

Questions I ran into as I was setting up and testing my first SQS queue were:

1. Does the queue need any sort of acknowledgement from the consumer that the message was processed?
2. How is it determined when a subsequent message in the queue will start being processed? Is there a time interval?
3. How are messages deleted from the queue?

Here are the answers. First, questions 1 and 3 are actually very related. SQS doesn’t require acknowledgements in the form of acknowledgement messages or status codes, but it *does* require action from the consuming resource— that resource must itself delete the message from the queue once it has finished processing it. This is how the consumer communicates to the queue that the message has been processed. In the case of lambda triggers, messages are deleted automatically upon the lambda’s completion. This is a function of AWS’ built-in integration between its Lambda and SQS services.

So, the credit engine lambda now receives the original Salesforce message by way of the SQS queue rather than from Salesforce directly. Finally, the credit engine runs— pulling transunion data, processing it, applying our business logic rules, and then sending the corresponding payloads to Salesforce.

Below is a diagram representing the flow of data through these components: 

*[TO-DO: create/ include sequence diagram]*

*However:* this solution alone didn’t seem to be effective. For some reason, multiple outbound messages were still getting sent from Salesforce to the publisher, all of which were forwarded on to the lambda via the queue. In retrospect, I should have focused my attention on *why* multiple outbound messages were still getting sent from Salesforce even though the publisher should have been handling the timely ack’s. Instead, I went for what seemed like a more expedient approach— focusing on the lambda’s idempotency. 

### Step 2: Idempotency

#### *Solution 2.a: Execution-in-Progress Flag*

In computer science, ‘idempotent’ describes the quality of a function or set of operations whose *effect* will be the same regardless of how many times it is run. 

A good example comes from HTTP methods. Consider the methods `PUT` and `DELETE`. When you run either of these operations, subsequent requests of that method type will not change the state of the data on the server. Once a record is deleted, it can’t be deleted again. Once a record is added, an identical record can’t be subsequently added. Therefore, these methods are considered idempotent. Methods like `POST` and `PATCH` however are not idempotent. Each time you send a post or patch request, the data on the server will be updated

MDN Web Docs summarizes the main tenet of idempotency as such: “To be idempotent, only the state of the server is considered.” *-[MDN Web Docs Glossary > Idempotent](https://developer.mozilla.org/en-US/docs/Glossary/Idempotent)*

As this relates to the credit engine lambda, it needed to be made idempotent such that multiple identical requests made to the lambda would have the same effect as a single request: *One* credit pull, returning *one* set of data and results.

Once again, my approach was a bit of a hack. I was feeling under pressure to get this API shipped, so rather than researching my own solution I borrowed a previously-used idempotency technique from elsewhere in our codebase. (I believe this experience has taught me not to cut that corner in the future.)

The basic strategy is this: 

- Create a `lambda-is-running` field in Salesforce (a checkbox on Opportunity records, in our case), and add it to the outbound message data
- At the beginning of the lambda, check the value of `lambda-is-running`
    - If False, proceed
    - If True, return (stop lambda)
- Update the `lambda-is-running` field to True before proceeding to the substantive code
- Run lambda (pull credit, perform data operations, send back to Salesforce)
- Update `lambda-is-running` field to False once the lambda has finished running

This approach is based on the logic that any identical requests sent to the lambda will fail the `lambda-is-running` check, and so will not be processed. In short: While a request is currently being processed, do not process any of its duplicate requests.

However, there was one important SQS default setting that I was unaware of at the time of implementing this that rendered this strategy ineffective as well. What I *thought* was happening was that each message that entered the queue would be processed independently by my lambda (more specifically, by threads of the lambda— since SQS processes messages in parallel, which requires it to run multiple instances of the lambda consumer at once). With this understanding, the visibility timeout would prevent each message from being consumed again before being fully processed, and each new instance of my lambda would query the Salesforce opportunity to get the current value of the `lambda-is-running` flag. Because the duplicate messages from Salesforce would be processed asynchronously all messages after the first one would retrieve a flag value of `True` and so terminate that run.

What was, in fact, happening is that messages were not being consumed and processed independently. As an [AWS Blog](https://aws.amazon.com/blogs/compute/understanding-how-aws-lambda-scales-when-subscribed-to-amazon-sqs-queues/) post explains it: “By default, Lambda batches up to 10 messages in a queue to process them during a single Lambda execution.” This means that —unless otherwise configured— the first 10 messages to enter a queue will be sent to *one* lambda instance as a batch, where they will be processed sequentially (one by one). This, combined with my visibility timeout setting had two very important implications for my message processing.

1. Because messages were processed sequentially, the `lambda-is-running` flag value would always return to `False` before the next execution. That’s why it had no effect in catching or mitigating duplicate requests. There was no scenario under this configuration where a message would be processed and the flag would be `True`.
2. Because my visibility timeout value was set based on the estimated completion time of *one* run of the credit engine lambda, it would obviously timeout before having processed all 10 messages in the batch. In the context of SQS, when a lambda execution fails to process the entire message or batch of messages it receives, it considers all messages unprocessed and returns them all to the queue. Therefore, on top of processing each message one-by-one, it would never succeed in processing the entire batch, so all 10 requests would be sent back to the queue (where they were available to be consumed and processed *again* 🤦‍♀️). 

After this setup failed, I also attempted to implement the same lambda-is-running check in the publisher itself, which also failed due to the nature of Salesforce outbound messages. In short, Outbound message retries are simply new messages *with the same data as the original outbound message* that are sent out until an ACK is received. This means that retries won’t contain the updated `lambda-is-running` value so, by definition, they will always send a value of False for this field. (The workflow rule triggering the outbound message includes the condition that the lambda-is-running flag must be False.)

#### *Solution 2.b: ID Lookup*

The second idempotency approach —and the final, successful solution— involved referencing a lookup table to determine whether credit had already been run on an applicant. The lookup table was a DynamoDb hash table, where its keys -or unique identifiers- were the ID’s associated with the applicant’s Opportunity record in Salesforce. 

In this setup, Salesforce messages that are sent to the publisher API contain applicants’ opportunity id, which is used by the publisher to search for that opportunity ID in the lookup table. If it is not found, this means credit has not been run for that application. In this case, the publisher adds that id to the lookup table, and then proceeds with passing the message to the SQS queue. 

If the ID *is* found in the table, this mans that that ID has already passed through the publisher and been added to the lookup table and sent to the SQS queue where it will be processed by the credit engine lambda. In this case we do not want to send that ID to the queue again, so the publisher prints a message indicating the message is a duplicate, and terminates without sending the message to the queue.

## Conclusion
The Critical Section Problem is an important concept to understand and be prepared to manage when architecting software. There are often many possible approaches to confronting this type of problem, and it is important to understand not just the theory behind these approaches and the code required to implement them, but the nuances of the specific tools employed as part of the implementation. This post has explored various solutions for addressing the CSP, as well as lessons learned in the implementation of each. I hope this review has provided a useful introduction to these concepts and tools, and instructive examples of things to watch out for and to research when implementing new tools or technologies.