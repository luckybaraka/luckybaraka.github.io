---
title: "Message Queues: Stop Making Your Users Wait"
date: 2026-05-28 00:00:00 +0000
categories: [Distributed Systems, Backend Engineering]
tags: [distributed-systems, message-queues, backend-engineering, system-design]
---

Let's imagine you have a service that does the following:

- Takes images from users (from the frontend)
- Resizes them, applies filters, and runs content moderation
- Saves the processed result to the database

Let's say each of those steps takes about 2 seconds. If the service is fully synchronous, the client waits a good 6 seconds before getting a response. What does that mean for the user? A long wait, or what we call **high latency**.

So how do message queues rescue us from this? Before we get there, let's be honest about what's wrong with the design.

---

## The problem with doing everything at once

### 1. Users are stuck staring at a spinner

Six seconds doesn't sound like much until you're the one waiting. A design like this isn't human-centred. You can lose money and earn it back, but you can't get back lost time. If you don't manage time well, someone or something else will manage it for you: your boss, your procrastination, and certainly an application that was never designed to respect the user's time.

### 2. One failure wipes out all the work

Say resizing finished fine, then filtering fails or times out. The whole request fails. The user gets an error, retries, and the resizing work we already did is thrown away. We start from zero.

### 3. Traffic spikes knock the system over

Now imagine traffic suddenly jumps from 50 requests a second to 5,000, maybe because of a campaign, a feature launch, or the end of the month when everyone submits at once. If your servers can only handle about 200 a second, everything above that times out or errors. The system doesn't slow down gracefully. It falls over.

---

## What a message queue actually is

A message queue is a **buffer that sits between the service that creates work and the service that does the work**.

- The **producer** creates the work. In our example, that's the upload service.
- The **consumer** does the work. That's a pool of workers that resize, filter, and moderate.
- The **queue** holds messages until a consumer is ready for them.

Here's the redesigned flow:

1. The user uploads an image.
2. The upload service saves the file and drops a small message on the queue: `"image 456 needs processing"`.
3. The upload service immediately tells the user, "Upload received."
4. Workers on the other side pick messages off the queue and process them in the background, in parallel.

Look at what happens to our three problems:

- **Latency:** the user gets a response in milliseconds, because the upload service only saves the file and writes a message.
- **Failures:** if a worker crashes, the message goes back on the queue and another worker picks it up. One failure doesn't sink the whole request.
- **Traffic spikes:** the queue just gets longer. Work is delayed, not dropped.

The key property behind all of this is **decoupling**. The producer doesn't know or care who processes the message, or when. That means you can scale each side on its own. The upload service is lightweight, while the workers might need lots of memory or even GPUs. With a queue between them, you don't have to run expensive machines just to accept uploads.

The picture I keep in mind is a hospital reception desk. The receptionist registers you, hands your file to the triage queue, and moves on to the next patient. They don't walk you to the doctor and wait outside the room. The queue of files is what lets reception and the clinic work at their own pace.

---

## How queues work under the hood

### Acknowledgements: don't delete until it's done

If the queue deleted a message the moment a worker picked it up, and the worker crashed halfway through, that message would be gone forever.

So instead, the worker must send an **acknowledgement (ack)** once it finishes. Only then does the queue delete the message. No ack means the queue assumes the work wasn't done and delivers the message again.

### Stopping two workers from grabbing the same message

That creates a new question: while worker A is still processing (and hasn't acked yet), what stops worker B from picking up the same message?

Different systems solve this differently:

- **Amazon SQS** hides a message from other consumers for a set period called the *visibility timeout* (for example, 30 seconds). If the worker acks in time, the message is deleted. If not, it becomes visible again and someone else can retry it.
- **Kafka** gives each partition to exactly one consumer in a group, so two consumers never compete for the same messages.
- **RabbitMQ** uses prefetch limits and ack timeouts on each channel to control how many unacknowledged messages a consumer holds.

The mechanism differs, but the goal is the same: a message should only be actively processed by one consumer at a time.

---

## Delivery guarantees: the edge case that bites

Even with acks, there's a tricky case. A worker finishes the job successfully, then crashes **just before** sending the ack. The queue thinks the work never happened and delivers the message again. The work runs twice.

For resizing an image, that's harmless. For something like "pay this provider KES 5,000 for claim 789," processing twice means paying twice. This is where **delivery guarantees** come in.

### At-least-once (the one you'll use most)

Every message is delivered at least once, but possibly more than once. This means your consumers must be **idempotent**: processing the same message twice gives the same result as processing it once.

- "Set claim 789's status to `APPROVED`" is naturally idempotent. Running it twice changes nothing.
- "Add 1 to the provider's claim count" is not. Running it twice adds 2.

You make consumers idempotent either by designing operations to set a value instead of changing it, or by recording what you've already processed and skipping repeats:

```go
func handle(msg ClaimMessage) error {
    // Skip messages we've already processed
    if alreadyProcessed(msg.ID) {
        return nil
    }
    if err := processClaim(msg); err != nil {
        return err // no ack, so the message will be retried
    }
    return markProcessed(msg.ID)
}
```

In practice, `processClaim` and `markProcessed` should be in the same database transaction. Otherwise a crash between the two puts you back where you started.

### At-most-once

The message is removed as soon as it's picked up. If something goes wrong, it's simply lost. This is only acceptable for data where losing a little doesn't matter, like analytics events or metrics.

### Exactly-once

Every message is processed exactly one time. It sounds ideal, but true exactly-once processing is very hard to achieve in distributed systems. Kafka supports a form of it within its own ecosystem, with real trade-offs. Unless you can explain and defend the mechanism, **at-least-once with idempotent consumers** is the safer and more practical choice, and it's what most production systems actually run.

---

## When should you reach for a queue?

Look for these signals:

1. **The work can happen later.** Ask: does the user need this result right now? Sending an email, generating a report, or processing an upload usually doesn't need to block the response.
2. **Traffic is bursty.** A queue absorbs spikes and lets workers catch up.
3. **The two sides have different needs.** A lightweight API and a heavy processing job shouldn't have to scale together.
4. **You can't afford to lose work.** If a downstream service is down, the queue holds the messages until it comes back.

And one warning: **don't put a queue in the middle of a request that needs a fast answer.** If your requirement is a response in under 500 ms, adding a queue almost guarantees you'll miss it, and you now also have to figure out how to get the result back to the waiting client. Queues are for work that can wait, even if "later" is only a few seconds.

---

## The harder questions

### Scaling: partitions and consumer groups

A single queue can only handle so much. To go further, you **partition** it: split it into several independent sub-queues that can be processed in parallel.

On the other side, a **consumer group** is a pool of workers that share the partitions between them. With 6 partitions and 3 consumers, each consumer handles 2 partitions. Need more speed? Add consumers, up to a limit. You can't usefully have more consumers than partitions. A seventh consumer on six partitions just sits idle.

### Choosing the partition key

The partition key decides which message lands in which partition. It works a lot like a shard key in a database, and it affects two things:

**Ordering.** Messages with the same key always go to the same partition, and order is guaranteed within a partition. Suppose a provider submits a claim and then sends a correction to that claim. If those two messages land on different partitions, the correction could be processed before the original claim exists. Using the claim ID (or the member ID) as the key keeps related messages together and in order.

**Even distribution.** You want work spread evenly. If you partition by payer, and one payer handles most of the volume, that partition is slammed while the others sit nearly empty. That's a **hot partition**. A key with more spread, like the claim ID, avoids it.

The trade-off is that the key that gives you the right ordering isn't always the key that spreads load best. This decision deserves real thought.

### When producers outpace consumers

A queue doesn't fix a capacity problem. It only buys you time. If 300 messages arrive every second and your consumers can handle 200, the queue grows by 100 every second and never catches up.

What you can do:

- **Scale consumers** automatically based on queue depth, or add partitions.
- **Apply backpressure:** slow the producers down by rejecting new work or telling clients "we're busy, try again shortly."
- **At minimum, alert on queue depth**, so you find out before it becomes an outage.

### Poison messages and dead-letter queues

Some messages will never succeed, like a corrupted file or a malformed payload. Without limits, the consumer retries forever and wastes resources on a message that can't be processed.

The fix is a **maximum retry count**. After, say, five failed attempts, the message moves to a **dead-letter queue (DLQ)**, a separate place where failed messages wait for someone to inspect and fix them. The main queue keeps moving.

### What if the queue itself goes down?

Durable queues like Kafka write messages to disk and replicate them across several servers (called *brokers*). If one broker fails, another has a copy.

Kafka also keeps messages for a configurable retention period, even after they've been consumed. That means you can **replay** them. If your consumers were down for an hour, they pick up where they left off. If a bug in a consumer processed messages incorrectly, you deploy a fix and reprocess from an earlier point.

---

## Which queue should you pick?

- **Kafka:** a distributed streaming platform built for high throughput and durability. It scales with partitions, supports consumer groups, and keeps messages after they're consumed, so multiple groups can read the same data and you can replay. A strong default if you're unsure.
- **Amazon SQS:** fully managed, with no infrastructure to run. Standard queues give very high throughput with best-effort ordering. FIFO queues give strict ordering at lower throughput. Great when you're on AWS and want simplicity.
- **RabbitMQ:** a traditional message broker with flexible routing through exchanges and bindings. Useful when messages need to be routed in more complex ways.

---

## Wrapping up

Message queues let you stop making users wait for work they don't need to see happen. They decouple producers from consumers, absorb traffic spikes, and spread work across a pool of workers. But they're not magic. They come with their own questions about duplicates, ordering, capacity, and failure, and designing a good system means answering those questions on purpose.

If there's one thing to take away: **use at-least-once delivery, make your consumers idempotent, and choose your partition key carefully.**