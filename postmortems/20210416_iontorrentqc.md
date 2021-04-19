| <img src="/assets/dipi.png" alt="DIPI Badge" width="150">      | Data Integrity and Pipeline Integration (DIPI) Working Group |
| -------------- | ----------------- |
| **Event**      | `20210416_iontorrentqc`  |
| **Systems**    | `Majora`, `Elan`, `Phylopipe1` |
| **Detected**   | `2021-04-16 10:44`|
| **Resolved**   | `2021-04-17 16:14`|
| **Team**       | `SN` `RC`         |
| **Related issues** | [#55](https://github.com/COG-UK/dipi-group/issues/55), [#58](https://github.com/COG-UK/dipi-group/issues/58)  |

## Summary

Elan 20210415 contained 375 sequences from a recently added sequencing platform (Ion Torrent).
30 sequences from the Ion Torrent platform were incorrectly marked as QC PASS as no platform specific quality control rules were in place at the time.
The Ion Torrent platform was recently added to the Majora metadata specification (see [#46](https://github.com/COG-UK/dipi-group/issues/46)), but due to insufficient documentation, the quality control system for Majora was not updated with rules for handling Ion Torrent data.
The data integrity issue was dectected when two of these sequences went on to cause Phylopipe 1 to fail to calculate a quality metric, and terminated on 2021-04-16.

We have mitigated the issue by adding the required rules, preventing Majora from issuing quality control decisions without at least one test, and providing specific documentation for adding sequencing instruments to the system in future.
Integrity was restored in the 2021-04-17 dataset by removing the 30 failed sequences.

## Leadup

On 2021-04-12, `ION_TORRENT` was added to the sequencing endpoint API as a valid option for `instrument_make` (https://github.com/SamStudio8/majora/commit/b039aa5c3eac84acf6f12e51dde191d714a99720).
375 Ion Torrent sequences were subsequently registered, uploaded and processed for the first time by Elan a few days later on the 15th.

## Fault

As part of the daily processing, Elan uses the Majora API to request a quality control report to be generated for each new Published Artifact Group (PAG).
We apply slightly different quality rules to Illumina and Oxford Nanopore data (reflecting the different strengths and weaknesses of the platform).
These rules are encoded in Majora and applied based on the `sequencing_platform` metadata of the PAG (which is populated by the `instrument_make` from the sequencing process).
Upon encountering the new Ion Torrent data, there were no suitable tests to be applied and Majora skipped the available Illumina and Oxford Nanopore profiles.
Success is determined by the absence of failure, so all skipped Ion Torrent sequences were issued with QC PASS status by Majora on 2021-04-15.

## Impact

Majora processed 375 Ion Torrent sequences on 2021-04-15, marking them all as QC PASS without conducting any quality control tests.
Elan 2021-04-15 published 30 sequences downstream that should have failed basic QC.

## Detection

The issue was detected by RC when investigating the failure of Phylopipe 1. RC visually inspected a sequence and thought it contained too many Ns to be a valid sequence and raised the finding with SN.

## Response

SN checked the sequence on Majora and noticed the PAG had passed QC despite having `NA` for both the Illumina and Oxford Nanopore quality profiles.
SN had implemented the Ion Torrent change and immediately believed the issue to be related.

## Recovery

* Patch Phylopipe 1 to handle this error case
* Determine QC rules for Ion Torrent data
* Implement QC rules for Ion Torrent data
* Retrospectively apply QC rules for Ion Torrent data
* Purge QC FAIL Ion Torrent data from the downstream data set

## Timeline

### 2021-04-12
* `ION_TORRENT` added to `instrument_make`

### 2021-04-15
* Elan processes 375 `ION_TORRENT` sequences
* Phylopipe 1.0 responds to Elan completion signal and starts

### 2021-04-16
* `08:51` - Phylopipe 1 fails and emits a `@channel` warning to `#tael-stream`
* `10:44` - RC reports to SN the appearance of oddly low quality data in the Elan 2021-04-15 dataset
* `10:52` - SN confirms on Majora that the QC rules have been skipped, resulting in a default QC PASS
* `11:00` - RC handles patching Phylopipe 1 to filter out sequences that cannot generate quality metrics
* `11:04` - SN opens issue on DIPI https://github.com/COG-UK/dipi-group/issues/55
* `11:33` - SN has approval from NL and RM on QC configuration for Ion Torrent
* `11:48` - SN injects new QC configuration to Majora and issues a request for QC to be reperformed on 375 affected PAGs
* `12:24` - QC completes and reveals 30 sequences should have failed basic QC
* `12:59` - SN manually unlinks 30 affected sequences from the downstream dataset
* `13:12` - SN patches Majora to prevent the issuing of QC decisions without at least one test

### 2021-04-17
* `16:14` - SN confirms Elan 2012-04-17 no longer contains the 30 affected sequences and the incident ends


## Whys

* 30 sequences that should have failed QC were set to QC PASS by Majora
* ...because there were no QC profiles for Ion Torrent data in Majora
* ...because we forgot to consider setting QC rules for this new option
* ...because there is no developer documentation for extending Majora
* ...because there is nobody available to write it as a priority
* ...because we are not resourced to do this

## Technical cause

Majora defaulted to issuing QC PASS in the case where no QC tests had been performed.

## Root cause

There is insufficient resource within CLIMB-COVID to generate great developer documentation.

## Backlog check

Writing maintainer's documentation is a backlog task that might have prevented this issue from happening.

## Recurrence

This issue has not happened before, but SN notes that `PACIFIC_BIOSCIENCES` is a permitted `instrument_make` that has no QC profile.

## Lessons learned

* Some complex behaviours are hinged on simple fields in Majora, such as selection of quality control parameters from a sequencing instrument make. These should be documented as changes to the fields clearly have wider implications on systems.
* Improving resources to have someone who can contribute documentation would be a benefit to users and developers.

## Corrective actions

* [x] Patch Majora to prevent skipped quality tests passing by default https://github.com/SamStudio8/majora/commit/e8394fec12bc7a66e9db94f0de2f2b4e043dd3f9
* [x] Remove `PACIFIC_BIOSCIENCES` from `instrument_make` to prevent the same thing happening again with PacBio data https://github.com/SamStudio8/majora/commit/b83019e4f326b60820edcf62177eea48a7ddb09d
* [x] Improve developer documentation around the adding of sequencing options https://github.com/COG-UK/dipi-group/commit/c85974e156fb068237e50dd2bb33519a407a4d1a
* [x] Distribute this as a reminder of why we need to be resourced appropriately
