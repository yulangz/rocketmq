# Status

- Current State: Proposed
- Authors: [xdkxlk](https://github.com/xdkxlk)
- Shepherds: [zhouxinyu](https://github.com/zhouxinyu)
- Mailing List Discussion: [dev@rocketmq.apache.org](mailto:dev@rocketmq.apache.org)
- Pull Request: #PR_NUMBER
- Released: <released_version>

# Background & Motivation

## What do we need to do

- Will we add a new module? No.
- Will we add new APIs? No.
- Will we add a new feature? Yes.

## Why should we do that

**Are there any problems with our current project?**


Although we now support sequential consumption in pop mode, pop orderly has the following two problems now.

1. The handle obtained by pop orderly cannot be used to changeInvisibleTime. As a result, an important part of the overall consumption logic is missing.
2. Pop orderly lacks the notification mechanism. When there is a message that can be consumed, the long polling request that has been suspended cannot be woken up in time, resulting in message delay.

**What can we benefit from proposed changes?**


Solve the above problems.

# Goals

**What problem is this proposal designed to solve?**

- Support changeInvisibleTime for pop orderly
- Improve the timeliness of messages for pop orderly

# Non-Goals

**What problem is this proposal NOT designed to solve?**

# Changes

## Architecture

### Support changeInvisibleTime for pop orderly

The basic function of changeInvisibleTime of pop orderly is the same as the changeInvisibleTime of normal messages, which is to modify the invisible time of a certain message to the time specified by the user. However, when pop message orderly in batch, in order to ensure the order of messages, its performance is different from the changeInvisibleTime of normal messages.

After the next visibility time of a message has been changed to _T0_  by changeInvisibleTime

- If this message is not Acked, then after _T0_ , repeatedly consume this message and the messages after the queue where this message is located.
- If the message is Acked, then after Ack, we can continue to consume subsequent messages.

For example

<img src="https://user-images.githubusercontent.com/10397306/195077506-8b7762d2-7103-41f3-824d-c096a9a8d78b.png" width="1100px">

If we ack Msg1 and Msg3 but changeInvisibleTime for Msg2, we will reconsume messages from Msg2 after next visibility time.

We will use add a new field `offsetNextVisibleTime`in `ConsumerOrderInfoManager.OrderInfo` to to record the next visibility time of each offset.

For example, we have pulled three messages through pop orderly at 00:00:00 AM, and the invisible time is 10s. Now, there will be three records in `OrderInfo` 

<img src="https://user-images.githubusercontent.com/10397306/195077623-1c71d259-ad29-43dd-8c54-a0fc2d8c543e.png" width="630px">

Then, we Ack the first message, so the consumer offset will move to 1.

<img src="https://user-images.githubusercontent.com/10397306/195077741-888efeef-8b22-444c-81a1-e116d48a77d8.png" width="630px">

When we ack the third message, the consumer offset will not move. The offset will only move to the largest contiguous offset.

<img src="https://user-images.githubusercontent.com/10397306/195077833-0794db24-ba78-451f-b8e9-7ecf898605e6.png" width="630px">


Then, we change the next visibility time of the second message to 00:00:20 AM.

<img src="https://user-images.githubusercontent.com/10397306/195077888-6ca9226d-5a8b-4dc9-8ccc-278f21a404bb.png" width="630px">

If we not ack the second message, we will reconsume the second and thrid message after 00:00:20 AM.

If we ack the second message before 00:00:20 AM, the consumer offset will move to 3.

### Add notification mechanism for pop orderly

When the long polling requests of pop orderly hang on the broker, we need to wake up the hanging requests when there are messages available for consumption, so that the messages can be quickly pulled down by consumers. 

When a queue that is popped orderly, there will be an in-memory lock to ensure its concurrency and sequentiality. When the lock is released, we can try to wake up the long polling request on the queue. We can use an in-memory timer to record the time which the lock is released. When the time is up, check to see if there is a message that can be pulled, and if so, wake up the pending long polling request.

At the same time, when the user call popMessage, changeInvisibleTime, and ackMessage, it is also necessary to update the time which the lock is expected to be released.

## Interface Design/Change

- Method signature changes? Nothing specific.
- Method behavior changes?

The CHANGE_MESSAGE_INVISIBLETIME will support the handle obtained by pop orderly.

- CLI command changes? Nothing specific.
- Log format or content changes? Nothing specific.

## Compatibility, Deprecation, and Migration Plan

- re backward and forward compatibility taken into consideration? No
- Are there deprecated APIs? No
- How do we do migration? No

## Implementation Outline

We will implement the proposed changes in pull requests.

# Rejected Alternatives

## How do alternatives solve the issue you proposed?