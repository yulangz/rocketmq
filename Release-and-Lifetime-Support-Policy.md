### Context 
As we know, Oracle provides Support on Oracle Java SE products in the Oracle Lifetime Support Policy. For product releases after Java SE 8, Oracle will designate a release, every three years, as a Long-Term-Support (LTS) release. Java SE 11 is an LTS release. For the purposes of Oracle Premier Support, non‑LTS releases are considered a cumulative set of implementation enhancements of the most recent LTS release. Once a new feature release is made available, any previous non‑LTS release will be considered superseded. For example, Java SE 9 was a non‑LTS release and immediately superseded by Java SE 10 (also non‑LTS), Java SE 10 in turn is immediately superseded by Java SE 11. Java SE 11 however is an LTS release, and therefore Oracle Customers will receive Oracle Premier Support and periodic update releases, even though Java SE 12 was released, details could see from here[1].

In addition, due to the recent COVID-19 epidemic, our 4.8.0 release has been delayed, which has seriously affected community planning. Therefore, we propose a lifecycle release and support plan to support our growing number of cloud customers with faster and stable iterations.


We are planning to make a release every 3 months. A month before the release date, the release manager will cut branches and also publish (preferably on the wiki) a list of features that will be included in the release (these will typically be RIPs, but not always). We will leave another week for "minor" features to get in (see below for definitions), but at this point, we will start efforts to stabilize the release branch and contribute mostly tests and fixes. Two weeks after branch cutting, we will announce code-freeze and start rolling out RCs, after which only fixes for blocking bugs will be merged.

### Release Composition Phase Definitions
 * Major Feature - Feature that takes more than 2 weeks to develop and stabilize. Requires more than one PR to complete the feature.
 * Minor Feature - Feature that takes less than 2 weeks to develop and stabilize. Requires mostly one PR to complete the feature.
 * Stabilization - This phase would involve writing incremental system tests for features and fixing any bugs identified in the checked-in feature.
 * Feature Freeze - Major features should have only stabilization remaining. Minor features should have a PR in progress that can be checked in by a week. A release branch will be cut at this point.
 * Code Freeze - Development is stopped. Blocker bugs will be fixed after code freeze. The first RC will be created at this point.

we will strictly ensure that a release happens on a given date. For example, in the first release, we will decide to have a release by the middle of March and we will stick to it.  We will drop features that we think will not make it into the release at the time of feature freeze and also avoid taking on new features into the release branch. Trunk development can continue as usual and those features will be in the following release. Ideally, we would have started stabilization around this time. About two weeks before the release date, we would call for code freeze and code check-ins will be allowed only if any blocker bugs are identified. In a rare scenario, we could end up with a feature that passed the feature freeze bar but still fails to complete on time. Such features will also be dropped from the release at the end to meet the release deadline.

### What Is Our Lifetime Support Policy?
Given 4 releases a year and the fact that no one upgrades four times a year, we propose making sure (by testing!) that a rolling upgrade can be done from each release in the past year (i.e. last 4 releases) to the latest version.

## Who Manages The Releases?
As usual, a committer shall volunteer. If no committer volunteers, we'll cancel a release due to lack of interest.

## Schedule
The proposed schedule is shown below. We will do a release from March 2020. We will follow a 4 monthly schedule for Apache RocketMQ releases.


[1] https://www.oracle.com/java/technologies/java-se-support-roadmap.html