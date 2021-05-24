Status
-----------------
Current State: Discuss
      
Authors: RongtongJin, duhengforever  
  
Shepherds: vongosling

Mailing List discussion: users@rocketmq.apache.org; dev@rocketmq.apache.org     

## Background & Motivation

### Chaotic commit messages

Good commit messages should contain some contextual information. We can get the specific work of the commit from the commit messages, and we can use git instructions to quickly track related issues.

But observe the following commit record (taken from latest commit messages of RocketMQ)

```
7e756882 [ISSUE #1931] Remove duplicated doAfterRpcHooks logic
7953c4f5 [ISSUE #2044] Fix DefaultLitePullConsumerImpl NPE (#2059)
49a722f4 Merge pull request #2064 from wcc526/master
70da3e01 Merge branch 'develop' into master
a2aa6c18 Fix Fastjson version for RCE problem
3f37c851 [ISSUE #1988] Fix the issue that can not update messageDelay correctly with mqadmin updateBrokerConfig command
b3ec283c Merge pull request #2043 from wqliang/selectNamesrv
67b8a2aa Add @Override for RMQOrderListener.java (#1939)
f12cc81e Init english version of the README
cf20a26c Fix typo in README
ecdeb1b4 Merge pull request #2039 from lebron374/comment_fix_v1
41ce16bb Merge pull request #2040 from lebron374/npe_fix_v1
3245bd3c select a new namesrv when namesrvAddrChoosed not in new addr list
4feb8cae npe fix
fae6825f comment fix
```

We found the following problems

1. Many commit messages don't explicitly indicate what the code has modified, so we need to guess what it has done.

2. Some of the commit messages are associated with the issue on Github, while others are not, so it is difficult to trace the specific issue and pull request from a certain commit message.

3. The format of commit messages is not uniform.

It can be seen that the current commit messages of RocketMQ are quite confusing. We need some conventions to help the commit messages become clean and tidy, so as to help developers and release managers better track issues and clarify the optimization in the version iteration.

### Confused issue list and pull request list

Same as commit messages, the current issue list and pull request list on Github are also confusing, mainly showing the following aspects:

1. The community labeling issues and pull requests is confusing, for example, some milestones labels are on issues, some are on pull requests, and types and modules of issue label are unclear.

2. The naming is confusing. Although pull request naming rules are specified, some developers did not submit in the required format, issues and pull requests did not express a clear meaning, and there was a mixture of Chinese and English.

We need to further clarify the agreement to obtain a clear list of issues and pull requests so that the community can develop healthily.

### Irregular merging behavior

Committers has some irregular operations when merging pull requests, such as arbitrarily merging, merging pull requests to master branch, merging their own submitted pull requests and so on. We need conventions to solve the irregular merger behavior, and ensure that every pull request in the community is strictly approved to maintain the healthy development of the project.

### Easy to form release notes

When we have clean commit messages and clean issue and pull request list, it can help release manager to better clarify the changes that occur between iterations of the version, and release manager can quickly write release notes without much modification.

## Goals

- Clean commit messages
- Clean issue list and pull request list
- Standard community merge guidelines
- Release manager can quickly write clear and concise release notes

## Contributor Operation Conventions

### Commit Messages Format Conventions

Commit message format: <type>(<scope>): <body>

**Type**

This describes the kind of change that this commit is providing.

**feat(feature)**
**fix(bug)**
**docs(documentation)**
**style(formatting, missing semicolons, …)**
**refactor(function)**
**test(adding missing tests)**
**chore(maintain)**
 
**Scope**

A scope can be anything specifying the place of the commit change. For example log, remoting, RPC, client, console, plugin, storage, etc...
You can use * if there isn't a more fitting scope.


**Body**

This is a very short description of the change.

Use imperative, present tense: “change” not “changed” nor “changes”.
Don't capitalize the first letter.
No dot (.) at the end

### Pull Request naming conventions

Unified pull request naming format

Pull requests naming format: [ISSUE #{issue number}] body
For example

[ISSUE #2085] Support graceful shutdown for push consumer
[ISSUE #1879] GroupTransferService may be blocked by ResponseCallback in SYNC_MASTER mode
It should be noted that there is a space between ISSUE and issue’s digital number. There must also be a space between the square brackets and the body. The first letter of the body is capitalized.

Body content conventions

The pull request of the enhancement, test, code style, document, new feature should use the Verb-object structure as much as possible.

For example
[ISSUE #2088] Optimize RocketMQ client's stats of RT to make sense.
[ISSUE #2007] Upgrade fastjson version to prevent serious security problem.
[ISSUE #1976] Improve the security of the system topic operation.
[ISSUE #1689] Add interfaces to remove unused statsItem in BrokerStatsManager class.

The pull request of the bug type should directly describe the content of the bug and avoid writing long sentences.

Counter-example:
[ISSUE #1901] Fix the issue that create reply message fail when using request/reply mode.
[ISSUE #1906] Fix the issue that booleanConstantExpression might lead to class loading deadlock.

Correct example:
[ISSUE #1901] Create reply message fail when using request/reply mode.
[ISSUE #1906] The booleanConstantExpression might lead to class loading deadlock.

## Committer Operation Conventions

### Label Conventions

ISSUE Label Conventions

Each issue needs to be labeled. Generally, an issue needs to have two types of labels, including

**Type**
- new feature
- bug
- enhancement
- test
- code style
- doc
- rip
- question
- discuss
- wontfix
- duplicate

**Module**
- remoting
- admin
- broker
- client
- store
- namesrv

Note: In principle, only one type label is used for an issue. Don’t abuse the enhancement label.

Pull Request Label Convention

Pull requests only labels milestones (note: pull request does not label the label of the module and type), and committer determines which version of the pull request is merged. New features are generally merged in the larger version.
PMC members can label urgent label according to the urgency of the pull request. The urgent label is only used when the pull request needs to be repaired urgently.

### Merge Conventions

document, code style, test: one committer (or core contributor) approve

bug: not involve store, remoting, common, two committers (or core contributor) approve, involves store, remoting, three committers (or core contributor) approve.

enhancement: two committer (or core contributor) approve. if the pull request will cause compatibility issues, three committers approve are required.

new feature, rip: more than 4 committers (or core contributors) are required to approve.

The number of valid votes does not include the pr submitter himself
It is strictly forbidden to merge pull requests submitted by yourself

Github provides three ways to merge: create a merge commit, Squash and merge, and Rebase and merge.

`Create a merge commit` is easy to lose author information. The problem of `rebase and merge` is that if pull request submitter has multiple useless commit records, they will be merged into the trunk, and the format is difficult to standardize. 

Therefore, it is mandatory to use square and merge for merging, and the commit name must be the name of pull request. If the name of pull request is not standardized, you also need to help the submitter standardize pull request name.

Before merging, note that the target branch of pull request must be develop, not master branch. If the target branch of the pull request is wrong, you can notify the submitter to change the target branch, or help the submitter switch the branch.

## Release Manager Operation Conventions

Reference: http://rocketmq.apache.org/docs/release-manual

In principle, release notes can be screened out by milestone label of pull request (it's better to proofread with the commit records in case of missing milestone label), and specific release note entry can be written by pull request name, And release notes entry can be divided into five categories: new feature, bug, enhancement, test, document and code style.




