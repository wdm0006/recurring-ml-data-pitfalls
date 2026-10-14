Pitfalls for ML on Recurring Data
=================================

**A reference for data problems and limitations one may encounter when building a recurring machine learning or data system  Intended to supplement the comprehensive [guide to bad data](https://github.com/Quartz/bad-data-guide) with problems specific to recurring systems.**

Many data science resources exist for the analysis of static datasets, but often in the real world, solutions and systems need to work in an automated fashion on recurring data.  This presents a unique challenge, because when the system is being designed and built, you know that you don't have all of the data.  You will, by design, be getting new data every day, week or however often.  It turns out, this new and unseen data is often pretty messed up. 

In this document, I aim to outline some of the common challenges associated with building recurring data/ML systems, both in terms of the data itself, and how it is sent/received.

# Index

 * [New categories in categorical data](#new-categories-in-categorical-data)
 * [Changing notation for missing values](#changing-notation-for-missing-values)
 * [Changing datetime notation](#changing-datetime-notation)
 * [Schema and column drift](#schema-and-column-drift)
 * [Unable to send data in event driven fashion](#unable-to-send-data-in-event-driven-fashion)
 * [Only able to send data in event driven fashion](#only-able-to-send-data-in-event-driven-fashion)
 * [Can only email data](#can-only-email-data)
 * [Unable to dedupe across files](#unable-to-dedupe-across-files)
 * [Partial or truncated deliveries](#partial-or-truncated-deliveries)
 * [Late-arriving corrections and restated history](#late-arriving-corrections-and-restated-history)
 * [Mixed update intervals](#mixed-update-intervals)
 * [Inconsistent data latency](#inconsistent-data-latency)

# Descriptions

## New Categories In Categorical Data

Often times recurring data will include some kind of categorical variable with a large number of options.  Examples might be fault codes, cities, or colors. A common pitfall in designing recurring data systems around this sort of data, is that the number of categories isn't always known ahead of time.  Depending on how that data is produced, you could see only red, blue, and green for the first year, then finally see a rare yellow.  If your system cannot handle in some way (by ignoring or accommodating) previously unseen categories, it is fragile to these changes.

## Changing Notation for Missing Values

Over long periods of time, the systems producing, storing and eventually sending your system data will change.  In the best case, it is simply version changes in the same software, but occasionally, the systems themselves may change (as a company migrates from Oracle to SAP, for instance). In some of these cases, notation for missing values could change in formats like csv.  If the format of the data you are receiving is prone to this kind of vulnerability, be careful to accommodate many common notations for missing values, even if the data you've seen so far doesn't use them (yet).

## Changing Datetime Notation

Both the timezone and the format itself of timestamps is subject to change over time, surprisingly. A robust recurring system can identify when these changes happen and react accordingly.

## Schema And Column Drift

Between deliveries, the structure of a file can change even when the values look fine: columns get renamed, added, dropped or reordered.  A system that parses by position will quietly read the wrong data into the wrong field after a reorder, while one that parses by name may silently ignore a new column or turn a missing one into nulls.  Neither fails loudly, so the damage shows up later as degraded results.  A robust recurring system compares each delivery's columns against what it expects and flags structural changes rather than guessing.

## Unable to Send Data in Event Driven Fashion

Many times an individual or IT department will either only be able to send data with a certain interval (nightly at midnight), or the data itself will only be refreshed at a certain interval (every 5 minutes).  A lot of the time this is just because somewhere in their pipeline, it comes down to a cron job.  This limitation can cause issues with any analysis that depends on some kind of time-variant event (e.g. a batch, or trip)

## Only Able to Send Data in Event Driven Fashion

Conversely, in some cases data can only be sent irregularly, like at the end of a trip or batch.  This may be because there is no direct pipe from the data generators and the central store, so that update only happens at certain times.  This can be an issue for recurring time dependent analyses.

## Can Only Email Data

A surprising number of companies, to avoid dealing with the redtape of opening ports or involving IT, will insist on emailing data in 25MB chunks.  Be able to accept data via email.

## Unable to Dedupe Across Files

In many cases, a customer will send a file every so often (say, daily), which may contains records from the previous period's file.  Each individual file has no duplicates, but the union of the two files does have duplicates.

## Partial Or Truncated Deliveries

A scheduled file or batch can arrive with fewer rows than the source holds: an upload cut off midway, a timeout, a missing partition or one shard that never landed.  The file parses cleanly and the columns are right, so nothing errors and it is ingested as if complete.  A missing day is also indistinguishable from a quiet day, so a gap in the data looks like a real signal.  A recurring system ends up silently training or scoring on the hole.  Completeness needs a signal independent of the file itself, such as expected row counts from the sender, manifests or checksums, or row-count bands drawn from history.

## Late-Arriving Corrections And Restated History

Some sources revise records they have already delivered, through corrections, backfills or restatements.  Unlike the duplicates above, the keys match but the values have changed, so a naive dedupe may keep the stale row or the wrong one.  A recurring system needs a rule for which version wins, and ideally keeps what was known as of each delivery.  Otherwise, retraining on the latest data quietly rewrites history, and a backtest or earlier model can no longer be reproduced or fairly evaluated.

## Mixed Update Intervals

Often, different source systems will have different update intervals.  So one file may be updated daily, while another is only weekly.

## Inconsistent Data Latency

Different source systems have differing amounts of latency.  So one daily file may contain at most recent data from one hour ago, while another may be at best 8 hours old.

Contributing
============

I'm sure that I've missed many points here.  If you know of any, or want to help elaborate on those that are here, gimme a pull request.

## License — [Creative Commons Attribution 4.0 International](LICENSE)
