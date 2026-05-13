# RFC 0056 - Improving RelEng Deployment Velocity & Confidence
* Comments: [#56](https://api.github.com/repos/mozilla-releng/releng-rfcs/issues/56)
* Proposed by: @bhearsum

# Summary

We will speed up the rate at which we deploy changes to RelEng systems and improve our confidence around deployments. We will aim to deploy weekly and have automatic monitoring and rollback of deployments. This may pave the way for future improvements (eg: continuous deployments), but those are explicitly out of scope for the time being.

## Background & Motivation

A few years ago deployments to our systems happened infrequently and generally only in response to specific necessary changes. This, combined with the lack of regular dependency updates, meant that the deployment process was fragile and any given deployment had a relatively high likelyhood of causing bustage.

We have since addressed these things: dependency updates happen regularly and predictably, and deployments for most systems happen at least every 2 weeks. These changes have been a huge success. Our deployment process is very reliable these days and bustage is exceedingly rare. We've even gotten to a point where we feel comfortable regularly doing off-cycle deployments to pick up some changes more quickly.

It is also notable that in one place, we're already well beyond this: Firefox-CI configuration changes are deployed on a continuous basis (ie: as they merge to `main`). While this is a simpler process than deploying backend systems, it suggests that we have the maturity and experience to start working towards this elsewhere.

# Details

We will enhance our tooling and automation to increase confidence in our deployments by:

1. Migrate the system to MozCloud
2. Develop smoketests that can validate a deployment
3. Automatic deployment to non-production environments when changes hit `main`
4. Use a more sophisticated rollout strategy
5. Automatic rollback of bad deployments

These improvements do not block moving to weekly deployments, but if we find that bustage increases from deploying more quickly we will reduce the frequency until the necessary tooling is in place.

## Scope

This proposal is mainly aimed at the systems that are well understood and well maintained: Balrog, Ship It, Scriptworkers, Tooltool, k8s-autoscale, and Firefox-CI.

Other systems we own that aren't well maintained are explicitly out of scope: buildhub, delivery dashboard, pollbot.

## Necessary Improvements

### Migration to MozCloud

Most of the improvements we need to make depend on our ability to make changes to the deployment and rollout procedures. MozCloud provides more options and flexibility in this regard. For this reason, we should build these improvements on top of it rather than trying to hack them into the existing Jenkins pipeline (which is deprecated anyways).

### Develop smoketests

It is crucial that we are able to quickly and confidently validate our deployments. The best way to do this will be to develop sets of smoketests that exercise all key functionality of a system. The exact details of these will vary from system to system, but the following principles must be kept in mind to ensure the right coverage is provided:

* All critical functionality must be tested
* Smoketests are a type of integration test that run against live systems - they are not unit tests. Other things a system depends on should _not_ be mocked out or otherwise worked around.
* When testing functionality that makes writes, smoketests must _not_ be given access to affect "real" data. eg: they must only make changes to test objects.

Especially when testing against production, it may be necessary to get creative in how functionality is exercised without impacting user or developer facing things. For example, many scriptworkers publish data to production systems (balrog, product delivery, etc.) - we will need to find ways of doing this without publishing real objects or data. One way of doing this could be to allow smoketests to write to sandboxed area of production systems (a non-published product in Balrog, a hidden directory on archive.mozilla.org, etc.).

It is noteworthy that scriptworkers have [some form of canary testing already](https://searchfox.org/firefox-main/rev/923c4d7d35ebb5693f5bda5dec9083f7c4f993b3/.cron.yml#418-423). These may provide some inspiration and a starting point for smoketests for scriptworkers, but we should not consider them to be enough. Eg: they are unable to do smoketesting on L3 pools.

### Automatic nonprod deployments

Deployments to nonprod environments should happen automatically when changes are merged to `main`. These deployments should have smoketests kicked off automatically to validate them.

This is a relatively low risk way to exercise our deployment pipeline more regularly, and get additional user testing, eg: through certain types of Try pushes.

### More sophisticated rollouts

At the moment most of the rollouts we do for our systems flip all traffic and usage over to new deployments instantly. We mostly rely on manual testing in nonprod environments to catch issues ahead of time. This is good enough for scheduled deployments, but with more frequent deployments the testing burden becomes higher, and automating it will both save time and increase confidence.

Generally, we should look at using canary-style rollouts. Compared with other common options (eg: blue/green) these allow us to gradually ramp up traffic instead of cutting it over instantly.

Canary rollouts should be properly integrated with smoketesting and automatic rollback:

* Smoketesting should be performed before _any_ real traffic is moved to the canary (more on that below).
* Automatic monitoring and rollback should be performed from the moment the canary deployment begins, until some amount of time _after_ full production traffic has been moved to the new deployment.

For backend services, smoketesting prior to the canary receiving traffic can be achieved by having a separate DNS entry that always routes traffic to the canary deployment, or sending a specific HTTP header for the load balancer to detect.

For scriptworkers, we will need some k8s-autoscale changes to cope with having multiple deployments that could be scaled up. Even with this done, canaries will necessarily be a best effort thing to do. Because scriptworker load is bursty, canaries will not be guaranteed to pick up many, if any, tasks. If this becomes problematic at any point, we will consider ways in which we can improve canaries or otherwise improve confidence in scriptworker deployments. (Under this plan, we will already have proper smoketests for scriptworkers, so we'll be in a much better position than we are today.)

### Automatic rollback

When new deployments happen, they should be monitored for a period of time, and automatically rolled back under certain conditions. At the very least, rollback should happen if any of the following are true:

* Smoketests fail
* Pods from the deployment crash
* New errors are found in Sentry
* System-specific errors spike (for example: 4xx and 5xx responses increase; tasks begin to fail)

Automatic rollback does not prevent us from manually rolling out the new version if we choose; it merely halts any further impacts until we have a chance to evaluate the situation and decide if we need to back out a change, make a fix, accept the new error rate, etc.

The automatic rollback period must also expire after a certain period of time to ensure that factors such as changes to traffic or usage that cause errors unrelated to the deployment do not rollback deployments.

# Open Questions


# Implementation

Tracking bug: https://bugzilla.mozilla.org/show_bug.cgi?id=2043561
