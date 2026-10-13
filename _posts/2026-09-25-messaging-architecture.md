---
title: "Messaging Architecture With NATS"
date: 2026-09-28 00:00:00 +0000
categories: [Distributed Systems, Backend Engineering]
tags: [distributed-systems, nats, jetstream, message-queues, backend-engineering, system-design]
---

Imagine a service that accepts images from users on the frontend, resizes them, applies filters, runs a content moderation check, and then saves the finished result to the database. Suppose each of those steps takes about two seconds. If the service does all of this synchronously, the person who uploaded the image waits a full six seconds before anything comes back to their screen. That waiting time is what engineers call high latency, and it is one of the first things users notice and one of the last things they forgive.

Message queues exist to rescue us from this situation. In this post I want to explain what they are, how they work, and how to think about the hard questions they raise. I will spend most of the time on NATS, because it is the messaging system we use every day on our team, and I will walk through how we use it to move fingerprint captures between our services and the hardware sitting on a clinic desk.

## What goes wrong when everything happens at once

The first problem is that the user is left staring at a spinner. Six seconds does not sound like much until you are the one waiting, and a design that makes people wait for work they do not need to watch is not a human-centred design. You can lose money and earn it back later, but nobody gets back the time they lose. If we do not design our applications to respect people's time, the application ends up managing their time for them, and it does it badly.

The second problem is that a single failure wipes out all the work that came before it. If resizing finishes successfully and then the filter step fails or times out, the whole request fails with it. The user sees an error and tries again, and the resizing that already succeeded is thrown away, so the system starts again from nothing.

The third problem is what happens when traffic spikes. Traffic might jump from fifty requests a second to five thousand because of a campaign, a new feature, or the end of the month when everyone submits their claims at once. If the servers can only handle around two hundred requests a second, everything above that number times out or fails. The system does not slow down gracefully under the extra load. It simply falls over.

## What a message queue actually is

A message queue is a buffer that sits between the part of a system that creates work and the part of the system that does the work. The service that creates the work is called the producer, which in our example is the upload service. The services that do the work are called consumers, which here would be a pool of workers that resize, filter and moderate images. The queue in the middle holds each message safely until a consumer is ready to take it.

With a queue in place, the flow looks very different. The user uploads an image, and the upload service saves the file and places a small message on the queue saying that image 456 needs processing. The upload service then tells the user straight away that the upload was received, while workers on the other side of the queue pick up messages and process them in the background, often several at the same time.

All three of our problems improve with this one change. The user gets a response in milliseconds, because the upload service only has to save a file and write a short message. If a worker crashes in the middle of a job, the message goes back on the queue and another worker picks it up, so one failure no longer ruins the whole request. When traffic spikes, the queue simply grows longer for a while, which means work is delayed instead of being dropped.

The property that makes all of this possible is called decoupling. The producer does not know or care which worker will process its message, or exactly when that will happen. Because the two sides do not depend on each other directly, each one can be scaled on its own terms. An upload service is lightweight, while the workers might need a lot of memory or even GPUs, and with a queue between them you never have to pay for expensive machines just to accept uploads.

The picture I keep in my head is a hospital reception desk. The receptionist registers you, places your file in the triage queue, and turns to the next patient in line. They do not walk you to the doctor and wait outside the consulting room until you come out. That pile of files is what allows the front desk and the clinic to work at their own pace without holding each other up.

## Why we chose NATS

NATS is an open source messaging system that is designed to be small, fast and simple to run. The server is a single lightweight program, it starts in a moment, and it can be clustered across several machines when you need more capacity or resilience. For a team that cares about operational simplicity, that matters a great deal, because every piece of infrastructure you run is something that can break at three in the morning.

The second reason is that NATS works comfortably across every language we use. Our orchestration service is written in Go, our biometrics middleware is written in Python with Django, and the hardware server that talks to the fingerprint scanner is written in C# on .NET. NATS has mature clients for all of these, and it can even reach the browser over WebSockets, so all of our services speak to each other through one shared message system instead of a patchwork of direct connections.

The third reason is that NATS gives you two levels of service from the same server. Core NATS is extremely fast publish and subscribe messaging that keeps nothing on disk. JetStream is a persistence layer built into the same server that stores messages, tracks acknowledgements, and redelivers anything that was not processed. You can start with the simple layer and switch on the durable layer only for the flows that genuinely need it.

## The core ideas in NATS

Everything in NATS is organised around subjects. A subject is simply a name that a message is published to, written as words separated by dots, much like a file path. We name our domain events in a pattern such as an entity followed by an action, so that a newly created patient record is announced on a subject that clearly says a patient was created. Subscribers listen to subjects, and they can use wildcards to listen to a whole family of subjects at once. One wildcard matches a single word in the name, and the other matches everything that follows it, which means a service can choose to hear about every action on patients without having to list each action by hand.

The simplest pattern is publish and subscribe. A producer publishes a message to a subject, and every subscriber listening to that subject receives a copy. This is ideal when several services each need to react to the same event in their own way, such as one service updating a dashboard while another writes an audit record.

When you want work to be shared out instead of copied, NATS uses queue groups. Several workers subscribe to the same subject as members of the same queue group, and NATS delivers each message to only one member of the group. This is how you get a pool of workers splitting the load between them, and adding capacity is as simple as starting another worker that joins the group.

NATS also supports request and reply. A service publishes a request and waits for a single answer, while NATS takes care of routing the reply back to the right caller. This feels like a normal remote call, but it still travels through the message system, which means the caller does not need to know the network address of whoever answers.

## JetStream, where messages become durable

Core NATS delivers a message to whoever is listening at that moment, and if nobody is listening, the message is gone. That is perfect for live updates where a missed message does not matter, but it is not acceptable when losing a message means losing a patient's fingerprint capture. JetStream exists for those cases.

In JetStream, a stream captures and stores every message published to a chosen set of subjects. The stream can keep messages on disk and can copy them across several servers in a cluster, so the data survives if one server goes down. A consumer is a view over that stream that tracks which messages have been delivered and which have been acknowledged. Consumers can be durable, which means they remember their position even when the service reading from them restarts, so a worker that comes back after a crash continues from where it stopped instead of starting over.

Consumers come in two styles. A push consumer has messages sent to the subscriber as they arrive. A pull consumer waits for the worker to ask for the next batch of messages. We prefer pull consumers for most work, because they let each worker take only as much as it can handle, which becomes important when we talk about pressure on the system later in this post.

## Acknowledgements and redelivery

If a queue deleted each message the moment a worker picked it up, and that worker crashed halfway through the job, the message would be lost forever. To avoid this, a worker must send an acknowledgement once it has finished, and only then does the system treat the message as done. If no acknowledgement arrives, the system assumes the work was never completed and delivers the message again.

This raises a second question, which is what stops another worker from grabbing the same message while the first worker is still busy with it. JetStream handles this with a setting called the acknowledgement wait. When a message is delivered, JetStream gives the worker a set amount of time to acknowledge it, which is thirty seconds by default. During that window the message is considered in progress and is not handed to anyone else. If the window passes without an acknowledgement, JetStream delivers the message again.

Other systems solve the same problem in their own way. Amazon SQS hides a message from other consumers for a visibility timeout, which works much like the JetStream acknowledgement wait. Kafka gives each partition to exactly one consumer in a group so that consumers never compete for the same messages. RabbitMQ limits how many unacknowledged messages each consumer can hold at once. The mechanisms differ, but every system is trying to make sure that only one consumer is actively working on a given message at any moment.

JetStream also lets a worker be more expressive than a simple yes. A worker can tell JetStream that it is still working and needs more time, which resets the clock on a long job. It can send a negative acknowledgement, which asks for the message to be redelivered, optionally after a delay. It can also mark a message as terminated, which tells JetStream to stop trying because the message can never succeed. These small signals make a real difference to how gracefully a system handles slow and failing work.

## How we move a fingerprint capture over NATS

Here is where the ideas above stop being theory. On our team, a clinic workstation has a fingerprint scanner connected to a small hardware server that runs on the same machine. When a partner application needs a fingerprint for a patient, the request travels through three of our services before it reaches that scanner, and NATS carries the most important part of the journey.

The flow starts when the partner's application calls our orchestration service over HTTP and asks for a fingerprint capture for a particular patient. The orchestration service does not hold that HTTP request open while it waits for a nurse to place a finger on a scanner. Instead, it records a new job, replies immediately with an HTTP 202 response and an operation identifier, and lets the caller check the progress of that operation whenever it likes. The 202 status code exists for exactly this case, because it tells the caller that the request has been accepted but the work is not finished yet.

The command is then published over NATS on a subject that names the exact workstation it is meant for, using a pattern that combines the purpose of the message, the hardware service, the workstation identifier, and the action being requested. Putting the workstation identifier into the subject means each hardware server only listens to commands addressed to itself, and NATS does the routing for us without any lookup table in our own code.

Every message we send is wrapped in a standard envelope. The envelope says what kind of message it is and what action it describes, carries the job identifier and the workstation identifier, includes an idempotency key, and sets a deadline after which the command should no longer be acted on. It also carries a privacy scope that labels how sensitive the contents are. Our scopes range from public, through general internal use and private data that identifies a person, up to restricted data such as clinical and biometric information. Biometric data is about as sensitive as data gets, so having its sensitivity written on the envelope means every service that touches the message knows how carefully it must be handled.

When the hardware server receives the command, it drives the scanner, collects the capture, and publishes the result back over JetStream. A consumer in our middleware picks up that result, checks it, and updates the job, and the next time the partner checks the operation, it sees the outcome.

We chose a message system for this flow because a clinic workstation is not always reachable on the timetable of an HTTP request. The machine might be busy, the network at a rural facility might be slow, or the nurse might still be calming the patient down. With NATS in the middle, the command waits for the device and the result comes back as an event, so the partner's application never has to depend on the scanner being ready at the exact moment it asks.

There are two lessons from this flow that I think apply far beyond biometrics. The first is that we separated two different questions that callers were mixing up. One question is what happened to the delivery of the command, and the other is what happened to the fingerprint itself. A clean capture that we failed to deliver is a problem for our engineers to fix, while a finger that will not read is a problem for the person at the desk to handle. Because those two answers belong to different people, we gave them two separate fields instead of forcing both into one status.

The second lesson is that messages can go missing in ways that nobody notices. At one point a validation failure meant that a job could sit in the dispatched state forever, because the result it was waiting for was silently rejected. We now run a sweep that looks at any job whose deadline has passed and checks its real state directly, so no job is left waiting indefinitely for a message that will never arrive.

## Delivery guarantees, and why idempotency matters so much

Even with acknowledgements, there is a tricky edge case that every messaging design has to face. A worker can finish a job successfully and then crash just before it sends the acknowledgement. The system believes the work never happened and delivers the message again, so the work runs twice. For resizing an image that is harmless, but for a message that says to pay a provider five thousand shillings for a claim, processing it twice means paying twice.

This is why messaging systems talk about delivery guarantees. Core NATS on its own gives you at most once delivery, which means a message reaches each listener once or not at all. That is acceptable for live dashboards or metrics, where losing an occasional data point does no harm, but it is not acceptable for anything you cannot afford to lose.

JetStream gives you at least once delivery, which means every message is delivered at least one time but may occasionally be delivered more than once. This is the guarantee most production systems run on, and it comes with one firm requirement, which is that your consumers must be idempotent. An idempotent operation produces the same result whether it runs once or several times. Setting a claim's status to approved is naturally idempotent, because running it again changes nothing. Adding one to a provider's claim count is not idempotent, because running it twice adds two.

There are two practical ways to make consumers idempotent. The first is to design operations so that they set a value instead of changing it relative to what was there before. The second is to record which messages you have already processed and skip any message you have seen before. When you use the second approach, the work itself and the record that it was done should be saved in the same database transaction, because if a crash happens between the two, you are back to processing the message twice.

JetStream can also get you close to exactly once processing, which is the guarantee everyone wishes they had. When a producer publishes a message with a unique message identifier header, JetStream remembers that identifier for a configurable window, which is two minutes by default, and quietly ignores any repeat of the same message inside that window. On the consumer side, a worker can wait for JetStream to confirm that its acknowledgement was received before moving on. Used together, these two features remove most duplicates, but they still depend on configuration and timing windows, so I treat them as a strong safety net and still build consumers to be idempotent.

In our capture flow, every command carries that unique message identifier, and the envelope carries its own idempotency key as well. If a partner sends the same capture request twice, our service returns the original job instead of creating a new one. That one decision means a patient is never asked to scan their finger a second time because of a network retry they cannot even see.

## When a queue is the right choice

There are four signals that suggest a queue belongs in a design. The first is that the work can happen later, so it is worth asking whether the user really needs the result right now. Sending an email, generating a report or processing an upload rarely needs to hold up the response. The second is that the traffic is bursty, because a queue absorbs sudden spikes and lets workers catch up at a steady pace. The third is that the two sides of the work have very different needs, such as a lightweight API in front of heavy processing that should not have to scale in lockstep. The fourth is that you simply cannot afford to lose work, because a durable queue holds messages safely until a downstream service that has gone down comes back.

There is also one situation where a queue usually does more harm than good. If a request must be answered in under half a second, placing a queue in the middle of it will almost certainly break that promise, and you also have to solve the extra problem of getting the result back to a client that is still waiting. NATS request and reply is fast enough for many quick calls between services, but once JetStream persistence and background workers are involved, the design is meant for work that can wait, even if that wait is only a few seconds.

## Scaling the work

A single queue can only move so many messages, and many systems deal with this by splitting a queue into partitions. Kafka is the best known example. Each partition is an independent sub queue, and a consumer group shares the partitions between its members, so with six partitions and three consumers each consumer reads two partitions. The limit is that you cannot usefully have more consumers than partitions, because a seventh consumer on six partitions has nothing to read and simply sits idle.

NATS approaches scaling a little differently. For most workloads, you scale by adding more workers to the same pull consumer or queue group, and NATS shares the messages among them without you having to plan partitions in advance. When you do need strict ordering for a related set of messages, the subject itself becomes your tool. NATS can map subjects onto a fixed number of partitioned subjects using a deterministic rule, so that every message about the same entity always lands in the same place and is processed in order.

Whichever system you use, the choice of what to partition on deserves real thought, because it affects two things at once. The first is ordering. Messages with the same key go to the same place, and order is guaranteed only within that place. If a provider submits a claim and then sends a correction to it, and those two messages are split up, the correction could be processed before the original claim even exists. Keying on the claim or the member keeps those messages together and in the right order.

The second is how evenly the work is spread. If you partition by payer and one payer carries most of the national volume, that one partition is overwhelmed while the others sit almost empty, which is known as a hot partition. Keying on something with a wider spread, such as the claim itself, avoids that imbalance. The difficulty is that the key that gives you correct ordering is not always the key that spreads the load best, and finding the right balance is one of the more important design decisions in any messaging system.

## When producers outpace consumers

A queue does not solve a capacity problem. It only buys you time to deal with one. If three hundred messages arrive every second and your consumers can only process two hundred, the backlog grows by one hundred messages every second and will never catch up on its own.

The first response is to add capacity by running more workers, ideally with automatic scaling that reacts to how many messages are waiting. The second response is to apply backpressure, which means slowing down the producers instead of letting the backlog grow without limit. JetStream gives you a few ways to do this. Each consumer has a limit on how many messages can be outstanding without an acknowledgement, which is one thousand by default, so a slow worker cannot be buried under more work than it can finish. A stream can also be given limits on how many messages or how many bytes it will hold, and it can be told to reject new messages once those limits are reached, which sends an error back to the producer instead of silently piling up more work. That error is the signal that lets a producer tell its own callers that the system is busy and that they should try again shortly.

At the very least, you should alert on the number of messages waiting to be processed, so that your team learns about a growing backlog well before it turns into an outage.

## Poison messages and dead letter queues

Some messages will never succeed no matter how many times you try them, such as a corrupted file or a payload in the wrong format. These are known as poison messages. Without a limit, a consumer will keep retrying such a message forever, wasting resources and sometimes holding up the healthy messages behind it.

The usual fix is a maximum number of delivery attempts. In JetStream, you set a maximum delivery count on the consumer, and once a message has been tried that many times, JetStream stops delivering it and publishes an advisory event that describes the failure. Unlike some other systems, JetStream does not have a built in dead letter queue, so the common approach is to listen for those advisory events and copy the failed messages into a separate stream where a person can inspect them, fix the underlying problem, and replay them if needed. Meanwhile, the main stream keeps moving, and one bad message no longer blocks everything behind it.

## What happens if the messaging system itself fails

A messaging system is only as useful as its ability to survive failure. JetStream can store messages on disk and copy every stream across several servers in a cluster, commonly three, so that if one server goes down, another server still holds a full copy of the data and carries on serving it. The servers agree on the state of each stream using a consensus protocol, which is what keeps the copies consistent with one another.

JetStream also lets you decide how long messages are kept, and this choice changes how the stream behaves. A stream can keep messages until it reaches a size or age limit, which suits a history of events that several services might want to read. It can keep messages only while there are consumers interested in them. It can also behave as a work queue, where each message is removed as soon as it has been acknowledged, which is the closest match to a traditional job queue.

When messages are kept after they are consumed, you can replay them. If your consumers were down for an hour, they pick up exactly where they left off once they return. If a bug in a consumer processed messages incorrectly, you can deploy a fix and read the stream again from an earlier point in time or from a particular message number, which turns a painful clean up into a routine recovery.

## Testing a messaging system honestly

One of the most useful lessons we learned came from testing. For a long time, our tests for the orchestration service replaced NATS with a fake. The sending side was tested against a fake receiver, and the receiving side was tested against a fake sender. If someone renamed a field in the envelope or changed the subject format, every test still passed, because each side was only checked against its own imitation of the other.

In production, that kind of mistake is completely silent. The partner gets its 202 response, the job sits in the dispatched state, and a clinician watches a scanner screen that never changes. To close that gap, we wrote an integration test that starts a real NATS JetStream server, connects a stand in for the hardware workstation, calls the real HTTP endpoint to request a capture, and lets the real result handler process the reply. The test checks that the subject is exactly what the workstation expects, that the unique message identifier header is set, that the envelope contains every field the C# hardware server reads, that the payload arrives unchanged, that the job finishes in the right state, and that sending the same request again returns the original job instead of triggering a second scan.

To prove the test was worth having, we broke the code on purpose. Renaming one of the envelope fields made the test fail, and so did changing the subject format, and once the code was restored the test passed again in about three seconds. If you take one practical habit from this post, I would make it this one. Test the contract between your producers and consumers against a real broker at least once, because a mock that agrees with your code will happily agree with your mistakes too.

## Choosing a messaging system

Kafka is a distributed streaming platform built for very high throughput and long term durability. It scales through partitions, supports consumer groups, and keeps messages after they are consumed, so many groups can read the same data and you can replay history whenever you need to. It is a strong choice for large event pipelines, although it asks more of the team that runs it.

Amazon SQS is a fully managed queue with no infrastructure to operate. Standard queues offer very high throughput with best effort ordering, while FIFO queues offer strict ordering at a lower throughput. It is a sensible choice when you already live inside AWS and simply want a dependable queue.

RabbitMQ is a traditional message broker with flexible routing through exchanges and bindings. It is useful when messages need to be routed to different places according to fairly involved rules.

NATS sits in an interesting middle ground. It is light enough to run almost anywhere, fast enough for live request and reply between services, and with JetStream it offers durable streams, acknowledgements, redelivery, deduplication, replicas and replay. For a team like ours, with services written in several languages and a strong preference for keeping operations simple, that combination is what made it the right fit.

## Wrapping up

Message queues let you stop making people wait for work they do not need to watch. They separate the services that create work from the services that do it, absorb sudden spikes in traffic, and spread work across a pool of workers. They are not magic, though, and they bring their own questions about duplicates, ordering, capacity and failure that a good design has to answer deliberately.

NATS gives you a clear way to answer those questions. You get subjects for routing, queue groups and pull consumers for sharing work, JetStream for durability and redelivery, and message identifiers for removing duplicates. In our own work, those tools are what allow a request from a partner application to reach a fingerprint scanner on a clinic desk and come back safely, even when the network is slow or the device is busy.

If you remember only one thing from this post, let it be this. Use at least once delivery, make every consumer idempotent, choose how you route and partition messages with care, and test the contract between your services against a real broker.