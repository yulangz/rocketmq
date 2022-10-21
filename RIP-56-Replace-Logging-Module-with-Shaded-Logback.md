## Status
* Current State: Proposed
* Authors: [aaron-ai](https://github.com/aaron-ai)
* Shepherds: [lizhanhui](https://github.com/lizhanhui),[zhouxinyu](https://github.com/zhouxinyu)
* Mailing List Discussion: dev@rocketmq.apache.org
* Pull Request: #PR_NUMBER
* Released: <released_version>
## Background & Motivation
### What do we need to do
* Will we add a new module? -- Remove a module actually.
* Will we add new APIs? No.
* Will we add a new feature? -- Yes.
### Why should we do that
#### Are there any problems with our current project?

The existing logging module is unnecessary, and the code quality is a concern.
	
#### What can we benefit from proposed changes?

Remove redundant logging module and keep it always latest with logback.
## Goals
### What problem is this proposal designed to solve?

The existing logging just aims to avoid the confliction between different bindings of SLF4J, but the code quality is a concern and it migrate a lot of code snippets from the previous version of logback/log4j, which may contains vulnerabilities.
## Non-Goals
### What problem is this proposal NOT designed to solve?

Nothing specific.
## Changes
### Architecture

![image](https://user-images.githubusercontent.com/19537356/197122276-c3537479-82c8-4c5b-8a0c-437daa0cbb41.png)

We intend to shade logback to replace the existing logging module. To avoid the clash of configuration file, the shaded logback will use rmq.logback.xml/rmq-logback-test.xml/rmq-logback.groovy rather than original logback.xml/logback-test.xml/logback.groovy.

Especially, the key of system property and environment is also be shaded to keep the original logback and the shaded logback independent.

An independent repo will be established to release the shaded logback periodically. You can see an example from: https://github.com/aliyun-mq/rocketmq-logging (Not the ultimate version)
### Interface Design/Change

No interface changes involved.
### Compatibility, Deprecation, and Migration Plan

This RIP doesn't introduce any compatibility issues.
### Implementation Outline
## Rejected Alternatives

I think there is no other cadicate alternatives.