## Status
* Current State: Proposed
* Authors: [lizhanhui](https://github.com/lizhanhui)
* Shepherds: NA
* Mailing List discussion: 
* Pull Request:
* Released: 

## Background & Motivation

1. What do we need to do

* Will we add a new module?

  No

* Will we add new APIs?

  No

* Will we add a new feature?

No

2. Why should we do that

* Are there any problems with our current project?

  Yes. RocketMQ follows the paradigm of long connections. Aka, the connections will be multiplexed for multiple remoting commands. This mechanism works well for regular cases. Unfortunately, when it comes to the server-offline use case, clients are frequently interrupted, witnessing a number of errors, and exceptions before the offline servers are passively detected and removed.


* What can we benefit from the proposed changes?

  Servers may be brought offline without interrupting clients

## Goals

* What problem is this proposal designed to solve?

  Support graceful offline of brokers. The proposing solution is similar to [HTTP 2.0 GO AWAY frame](https://datatracker.ietf.org/doc/html/rfc7540#section-6.8) mechanism

* To what degree should we solve the problem?

  Scheduled broker restart shall not interrupt clients.

##  Non-Goals

* What problem is this proposal NOT designed to solve?
  NA

* Are there any limits to this proposal?
  No

##  Changes

### Architecture

* Interface Design/Change

Will not change APIs that are visible to users.

* Compatibility, Deprecation, and Migration Plan

* Are backward and forward compatibility taken into consideration?

  Yes. If the broker servers send the Go-Away command to old version clients, clients will simply ignore it. If new clients worked with old broker servers, no GoAway command is received and things fall back to the previous behavior, which is precisely the same as an abrupt server shutdown.

* Are there deprecated APIs?

  No

* How do we do the migration?

  Not applicable

## Implementation Outline

#### We will implement the proposed changes in 1 phase.
* Phase 1

   1. Introduce the concept of ManagedChannel, providing the flexibility of managing multiple underlying TCP connections
   2. Improve broker Servers, sending GoAway commands to clients when it is about to shutdown
   3. For clients, once a GoAway command is received, create alternative TCP connections and route new commands to them if possible; Otherwise, isolate the ManagedChannel and drain existing inflight commands as soon as possible.

## Rejected Alternatives 

* How will alternatives solve the issue you proposed?

  No alternatives are foreseen.

* Pros and Cons of alternatives

  Not applicable 

* Why should we reject the above alternatives

  Not applicable