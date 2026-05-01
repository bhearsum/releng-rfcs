# RFC 55 - Replace Update Verify
* Comments: [#55](https://api.github.com/repos/mozilla-releng/releng-rfcs/issues/55)
* Proposed by: @bhearsum

# Summary

Replace current "update verify" tests with modern, independent testing of MAR verification; get rid of the "liveness" test that it does entirely.

## Background

Update verify is a collection of bash and python scripts that verify aspects of the MARs that Firefox uses to bring itself up to date and the metadata in Balrog used to inform Firefox of which MAR to apply. These two checks are distinct, and described in more detail below.

### MAR Verification

The purpose of MAR verification checks is to ensure that MAR files get users to a functionally equivalent place as a fresh install. This is important for two reasons:
1) It ensures that users get a functional build with the same code that automated and QE tests have been run against.
2) It ensures that when users that apply this MAR update in the future, partial MARs will apply cleanly.

### Liveness Checks

The liveness check part of update verify ensures that the release metadata in Balrog contains entries for all necessary platforms, locale, and version combinations.

Notably, this testing is very similar to what we do in the `final-verify` test.

## Motivation

There are a number of reasons to consider modernizing and changing the way we perform these tests:
* Our current "update verify" tests represent a significant amount of time in the critical path of building releases. In betas, individuals tasks take close to an hour to complete and spent over 2 hours in end to end time to complete all update verify tasks. In releases, these numbers are 30 minutes and 1h45min. In both cases, the end to end time is a majority of the overall end to end time of the graph.
* It is the least understood code that we own and is architecturally unsound: it is a cobbled together set of bash and python scripts that call back and forth to one another. For this and other reasons, it is difficult to confidently hack on.
* Its output is poorly understood, often leading to delays in understanding and fixing failures.
* Update verify tries to do too many things at once, which is partly to blame for the above two points.
* In a review of its history, it has rarely caught issues that would hit users. In fact, it failed to catch [a very serious issue relatively recently](https://bugzilla.mozilla.org/show_bug.cgi?id=1890528).

# Details

## MAR Verification

Update verify will be replaced by a separate tool for MAR verifications. We will reduce the scope and extent of MAR verification due to the lack of evidence that supports the amount of testing of this type that we currently do.

With update verify going away, we will also get rid of `update-verify-config` tasks, and various support scripts and code that will no longer be used.

## Liveness checks

Update verify liveness checks will not be replaced; we will instead rely on `final-verify` to check liveness.

## MAR Verification

### Current Testing

MAR Verification is currently done with a combination of shell and python scripts. We test all locales for versions receiving a partial MAR, and a limited selection of locales for a limited number of older versions. These tests involve:
* Contacting Balrog to see which updates are being offered
* Downloading MARs and installers from archive.mozilla.org
* Applying MARs to `from` installers
* Diffing the result against `to` installers
* Verifying that no expected differences are found

Aside from downloads, all of this is done serially.

### Proposed Testing

We should move MAR verification to Marannon, which was [recently created to give us this sort of coverage for Nightly](https://bugzilla.mozilla.org/show_bug.cgi?id=1837440). Marannon has many benefits over the current scripts, including:
* It will eventually fully replace the existing scripts (it currently calls out to parts of update verify, but this will be replaced in time).
* It focuses exclusively on MAR verification.
* It does not depend on Balrog to perform tests; this means tests can be run earlier in release graphs and are fully idempotent.
* It runs tests in parallel, making it significantly faster than update verify for the same number of tests.

In addition to using Marannon for running these tests, we should reduce the amount of testing that we do as follows:
* Verify complete and partial MARS for all locales against `from` versions that have a partial generated for them. This ensures that all completes and partials are tested against recent versions.
* Verify complete MARs for all locales against the first version after the last watershed for each platform. This does not test any MARs that weren't tested above, but it does verify the complete MARs against the most disparate version to the current one.

The second point is a departure from `update-verify` in two ways:
* It would eliminate all MAR verification done between those two points. It is noteworthy that over the nearly 20 years we have run update verify, we have no evidence of ever having an issue that would be caught by such checks.
* We will be testing against all locales against the post-watershed version instead of just a selection of locales. Because of the tests we're eliminating between that version and recent versions we can do this while still having a net negative effect on the total number of tests run.

## Liveness Checks

### Current Testing

Update verify accomplishes this goal by by making requests to Balrog for each combination of these (plus each version going back to the watershed) and validating a few things:
* That MARs are returned by Balrog, and that the links to MARs returned point at real files (ie: are not 404s)
* Ensuring that the hashes and filesize in the Balrog response match the downloaded file.
* Verifying that the buildID, appVersion, and displayVersion match what is in the update verify config.
* Verifying that partials are smaller than completes.

The current implementation is less than ideal in a couple of ways:
* It goes further than is necessary by downloading MAR files (although in most cases these have been cached already).
* Requests to Balrog are done against the `localtest` channel, which is neither a live channel, nor one that returns the same data as a live channel.

### Proposed Testing

We should not replicate this part of update verify. `final-verify` already performs the necessary checks (that all Balrog links return MARs, and that MARs are not 404s) on the live channel in the `push` phase of a release. The only advantage of doing a check like this earlier is that we would find issues with the Balrog metadata earlier. In practice, issues of this type virtually never happens these days.

The extra checks that update verify does are not necessary, and there is no evidence that they have ***ever*** caught a real issue:
* The hashes returned by Balrog are not checked by Firefox itself anymore, so there is no point in doing so in our tests.
* buildID, appVersion, and displayVersion are mostly just metadata for display, and the Balrog data and update verify configs ultimately get their information from the same place anyways.

Checking that partials are smaller than completes is not covered at all by `final-verify`, which is an acceptable loss. If that check is something we desire to keep we should reimplement it to happen at partial MAR generation time instead.

While we won't need to make any changes to `final-verify` itself as part of this proposal we will need to mark is as blocking `release-balrog-scheduling` to ensure that Balrog metadata is verified before we ship a release. (It currently blocks no other tasks; so failures could conceivably go unnoticed before shipping.)

# Open Questions

<what isn't decided yet? remove this section when it is empty, and then go to
the final comment phase>

# Implementation

<once the RFC is decided, these links will provide readers a way to track the
implementation through to completion>

* <link to tracker bug, issue, etc.>
* <...>

