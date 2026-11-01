Pitfalls for ML on Recurring Data
=================================

**A reference for data problems and limitations one may encounter when building a recurring machine learning or data system  Intended to supplement the comprehensive [guide to bad data](https://github.com/Quartz/bad-data-guide) with problems specific to recurring systems.**

Many data science resources exist for the analysis of static datasets, but often in the real world, solutions and systems need to work in an automated fashion on recurring data.  This presents a unique challenge, because when the system is being designed and built, you know that you don't have all of the data.  You will, by design, be getting new data every day, week or however often.  It turns out, this new and unseen data is often pretty messed up. 

In this document, I aim to outline some of the common challenges associated with building recurring data/ML systems, both in terms of the data itself, and how it is sent/received.

# Index

 * [New categories in categorical data](#new-categories-in-categorical-data)
 * [Changing notation for missing values](#changing-notation-for-missing-values)
 * [Changing datetime notation](#changing-datetime-notation)
 * [Unit and scale changes in numeric fields](#unit-and-scale-changes-in-numeric-fields)
 * [Schema and column drift](#schema-and-column-drift)
 * [File encoding and delimiter changes](#file-encoding-and-delimiter-changes)
 * [Unable to send data in event driven fashion](#unable-to-send-data-in-event-driven-fashion)
 * [Only able to send data in event driven fashion](#only-able-to-send-data-in-event-driven-fashion)
 * [Can only email data](#can-only-email-data)
 * [Unable to dedupe across files](#unable-to-dedupe-across-files)
 * [Duplicate-delivery replays](#duplicate-delivery-replays)
 * [Partial or truncated deliveries](#partial-or-truncated-deliveries)
 * [Late-arriving corrections and restated history](#late-arriving-corrections-and-restated-history)
 * [Mixed update intervals](#mixed-update-intervals)
 * [Inconsistent data latency](#inconsistent-data-latency)

# Descriptions

## New Categories In Categorical Data

Often times recurring data will include some kind of categorical variable with a large number of options.  Examples might be fault codes, cities, or colors. A common pitfall in designing recurring data systems around this sort of data, is that the number of categories isn't always known ahead of time.  Depending on how that data is produced, you could see only red, blue, and green for the first year, then finally see a rare yellow.  If your system cannot handle in some way (by ignoring or accommodating) previously unseen categories, it is fragile to these changes.

## Changing Notation for Missing Values

Over long periods of time, the systems producing, storing and eventually sending your system data will change.  In the best case, it is simply version changes in the same software, but occasionally, the systems themselves may change (as a company migrates from Oracle to SAP, for instance). In some of these cases, notation for missing values could change in formats like csv.  If the format of the data you are receiving is prone to this kind of vulnerability, be careful to accommodate many common notations for missing values, even if the data you've seen so far doesn't use them (yet). See also Quartz: [Zeros replace missing values](https://github.com/Quartz/bad-data-guide#zeros-replace-missing-values).

## Changing Datetime Notation

Both the timezone and the format itself of timestamps is subject to change over time, surprisingly. A robust recurring system can identify when these changes happen and react accordingly. See also Quartz: [Date formats are inconsistent](https://github.com/Quartz/bad-data-guide#date-formats-are-inconsistent).

## Unit And Scale Changes In Numeric Fields

Between deliveries, the same numeric column can silently change units, currency, scale or rounding while its name and type stay the same.  Examples might be amounts switching from cents to dollars, weights from kilograms to pounds, or counts from thousands to individual units.  The values are still valid numbers and pass type and schema checks, so the damage may show up as degraded predictions rather than an error.  A robust recurring system treats units, currency, scale and precision as part of the data contract, and compares incoming ranges against history to flag unexpected changes. See also Quartz: [Units are not specified](https://github.com/Quartz/bad-data-guide#units-are-not-specified).

## Schema And Column Drift

Between deliveries, the structure of a file can change even when the values look fine: columns get renamed, added, dropped or reordered.  A system that parses by position will quietly read the wrong data into the wrong field after a reorder, while one that parses by name may silently ignore a new column or turn a missing one into nulls.  Neither fails loudly, so the damage shows up later as degraded results.  A robust recurring system compares each delivery's columns against what it expects and flags structural changes rather than guessing.

## File Encoding And Delimiter Changes

Between deliveries, the file format itself can change while the content stays the same: the character encoding moves from UTF-8 to Latin-1 or Windows-1252, a byte-order mark appears, the delimiter switches from comma to semicolon or tab, or the quoting, line endings or decimal separator change.  The first samples parse fine, and the change usually arrives with an exporter upgrade or a locale change on the sender's side.  Some of these fail loudly, but many do not: a new delimiter can collapse a row into a single column, a decimal comma can turn numbers into strings or shift digits, mojibake in labels shows up as new categories, and a BOM can glue itself onto the first column name.  A robust recurring system declares the expected encoding, delimiter, quoting and decimal separator in the data contract, checks each file against them, and fails on an unexpected parse shape rather than guessing. See also Quartz: [Text is garbled](https://github.com/Quartz/bad-data-guide#text-is-garbled).

## Unable to Send Data in Event Driven Fashion

Many times an individual or IT department will either only be able to send data with a certain interval (nightly at midnight), or the data itself will only be refreshed at a certain interval (every 5 minutes).  A lot of the time this is just because somewhere in their pipeline, it comes down to a cron job.  This limitation can cause issues with any analysis that depends on some kind of time-variant event (e.g. a batch, or trip)

## Only Able to Send Data in Event Driven Fashion

Conversely, in some cases data can only be sent irregularly, like at the end of a trip or batch.  This may be because there is no direct pipe from the data generators and the central store, so that update only happens at certain times.  This can be an issue for recurring time dependent analyses.

## Can Only Email Data

A surprising number of companies, to avoid dealing with the redtape of opening ports or involving IT, will insist on emailing data in 25MB chunks.  Be able to accept data via email.

## Unable to Dedupe Across Files

In many cases, a customer will send a file every so often (say, daily), which may contains records from the previous period's file.  Each individual file has no duplicates, but the union of the two files does have duplicates. See also Quartz: [Rows or values are duplicated](https://github.com/Quartz/bad-data-guide#rows-or-values-are-duplicated).

## Duplicate-Delivery Replays

A provider or transport can send the same file or event batch again: a retry after a timeout, at-least-once delivery, a re-run job or a manual resend.  Unlike the overlapping files above, the whole delivery is repeated, and it is byte-identical or nearly so, so it passes every format and schema check.  Nothing errors: counts, sums and training rows are inflated, and an unusual day gets double weight.  Recurring systems are exposed because the same delivery is expected to show up again on the next run, with no human watching to notice the second copy.  A robust recurring system makes loads idempotent by keying them on file or batch identifiers and content hashes, and checks a ledger of what has already been delivered before ingesting anything. See also Quartz: [Rows or values are duplicated](https://github.com/Quartz/bad-data-guide#rows-or-values-are-duplicated).

## Partial Or Truncated Deliveries

A scheduled file or batch can arrive with fewer rows than the source holds: an upload cut off midway, a timeout, a missing partition or one shard that never landed.  The file parses cleanly and the columns are right, so nothing errors and it is ingested as if complete.  A missing day is also indistinguishable from a quiet day, so a gap in the data looks like a real signal.  A recurring system ends up silently training or scoring on the hole.  Completeness needs a signal independent of the file itself, such as expected row counts from the sender, manifests or checksums, or row-count bands drawn from history. See also Quartz: [Data are missing you know should be there](https://github.com/Quartz/bad-data-guide#data-are-missing-you-know-should-be-there).

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
