| <img src="/assets/dipi.png" alt="DIPI Badge" width="150">      | Data Integrity and Pipeline Integration (DIPI) Working Group |
| -------------- | ----------------- |
| **Event**      | `20220105_badfasta` `20220105_unpublish`  |
| **Systems**    | `Elan` |
| **Detected**   | `2022-01-05 12:00`|
| **Resolved**   | `2022-01-07 10:30`|
| **Team**       | `SN`, `SW`        |
| **Related issues** | [#179](https://github.com/COG-UK/dipi-group/issues/179), [#178](https://github.com/COG-UK/dipi-group/issues/178), [#38](https://github.com/COG-UK/dipi-group/issues/38) |

## Summary

This report summarises two closely related incidents:

* `20220105_badfasta` -- Elan 20220105 contained 171 egregiously malformed sequences, which interrupted the smooth running of downstream pipelines.
* `20220105_unpublish` -- During the response to `20220105_badfasta` 13,986 valid sequences added on 20220105 were inadvertently removed from the data set and no new sequences were published on 2022-01-05.

The malformed data was forcibly suppressed from Majora and the `published_date` of the missing data was corrected on 2022-01-16 to ensure they would be published after the next run of Elan.
Integrity was restored when the missing sequences were published correctly on 2022-01-07.

## Leadup

* Wider discussions on what should be done with sequences with non-IUPAC characters had long stalled (https://github.com/COG-UK/dipi-group/issues/38) and was not a pressing concern as data has consistently been in the correct format. Unfortunately, this combined with the incredibly lenient parsing from `readfq` left Elan vulnerable to non-FASTA files being processed as sequence data.
* The ingested data was not sequence data and posed such a great risk to downstream pipelines, the decision was made to republish the data set for the day.
* Republishing the daily data set has never been done before.


## Fault

### `20220105_badfasta`
As part of daily processing, Elan performs a very basic check of ingested FASTA files. 
This `fasta_quickcheck` step uses Heng Li's lightweight `readfq` parser. Although the `fasta_quickcheck` script contained guards to check the quality of the parsed data, it underestimated the leniency of `readfq`. The files in question were correctly formatted with a FASTA header. Although the sequence data appeared to contain a manifest of some sort, as the files had a valid FASTA header and did not trigger any of the abort cases in the parsing script, they were forwarded for processing by Elan.

### `20220105_unpublish`
Since the "caffeine cat" update (https://github.com/SamStudio8/elan-nextflow/commit/d9b710a7a1ac1ed293a1513c22f5e471a49a1c9b), the daily artifacts are built through a combination of using the previous data set (referenced via a directory named `latest`) as a cache, and querying Majora for new Published Artifact Groups (PAGs) that have passed QC. Majora is also queried to find if sequences were suppressed since the previous data set was published; allowing it to filter suppressed data from the daily data set.

Since `latest` was upgraded from a symbolic link to a directory; a symbolic link named `head` is used to point to the most recent stable daily artifacts. `head` is updated at the last possible moment of the publishing pipeline, to make it possible to pick up the publishing pipeline after an error, even if artifacts or links in `latest` have been updated.
`head` is used to determine the threshold date for the Majora queries. That is, when publishing the data for day `i`, `head` is pointing to day `i - 1` and Majora is asked to enumerate the new and suppressed PAGs for days `> i - 1`.

When the command to republish the 20220105 data set was issued, the user correctly repointed `latest` to 20220104 manually; and crucially, forgot to update the `head`.
The publish script queried Majora for PAGs with a published date after (and not equal to) 2022-01-05. Majora returned zero groups and the publish step continued with 0 additions and 171 suppressions; essentially republishing the 20220104 data set.

## Impact

* Elan 20220105 contained no new sequences and was effectively a copy of Elan 20220104.
* Elan 20220106 did not contain sequences processed on 2022-01-05.

## Detection

* `20220105_badfasta` -- The malformed data allowed through by Elan 20220105 was detected by SN at `2022-01-05 12:00` after investigating why Asklepian 20220105 could not be re-raised by SW. SN diagnosed the malformed data through visual inspection.
* `20220105_unpublish` -- The missing data was detected `2022-01-06 AM` by a downstream user who was attempting to retrieve some new data that was expected to have been ingested as part of Elan 20220105.

## Response

* `20220105_badfasta` -- SN patched Majora to allow affected data to be suppressed without approval from the data provider and SW suppressed the affected sequences. SN used `cog-publish` to regenerated the 2022-01-05 data set from scratch.
* `20220105_unpublish` -- SN updated the `published_date` for the affected Published Artifact Groups (PAGs) to one day in the future, allowing them to be folded in to the next build of the data set.


## Recovery

### `20220105_badfasta`
* Identify the bad data
* Forcibly suppress bad data to maintain data set integrity
* Expedite removal of bad data by running the publish step for the day manually
* Raise Asklepian
* Improve the `fasta_quickcheck` process to stop this sort of embarrassment happening again

### `20220105_unpublish`
* Identify the missing data by querying for PAGs with a `published_date` of 2022-01-05
* Update the `published_date` of the affected sequences in Majora to 2022-01-07
* Prevent `cog-publish` from being run in such a disasterous way again

## Timeline

### 2022-01-05
* `11:20` - Asklepian terminated with exit 1 and alerted `#tael-stream`
* `11:26` - SW opens #178 (https://github.com/COG-UK/dipi-group/issues/178)
* `11:31` - SW attempted to re-raise Asklepian
* `11:31` - Asklepian terminated again with exit 1 and alerted #tael-stream
* `11:48` - SW updates #178 to escalate to SN
* `12:00` - SN on the scene
* `12:12` - SN determines the problem is down to malformed data accepted by Elan and affects 171 records
* `12:18` - SN outlines recovery plan on #178
* `12:56` - SN patches Majora to add the `can_suppress_any_pags_via_api` permission, granting users the right to suppress any PAG (https://github.com/SamStudio8/majora/commit/916c5d4c3d4559c74456ed85e2833c4a4b575260)
* `13:03` - SN live migrates Majora and grants `can_suppress_any_pags_via_api` to SW on production Majora
* `13:12` - SW finishes suppressing the 171 affected artifacts
* `13:30` - SN confirms suppression was successful with Ocarina `get pagfiles`
* `14:06` - SN repoints `latest` to `20220104`
* `14:10` - SN confirms the `cog-publish` script has run successfully with 171 suppressions
* `14:10` - The Elan publish pipeline updates `latest` to 20220105, containing a replica of 20220104
* `14:18` - SW successfully raises Asklepian
* `14:50` - SN confirms an Asklepian MSA is generated successfully
* `14:59` - SN approves patch from SW that will ensure Elan rejects sequences containing non-IUPAC characters going forward
* `15:11` - SN confirms Asklepian has begun generating the artifacts for output

### 2022-01-06
* `09:07` - Elan 20220106 is published, still missing data that was processed on 2022-01-05
* `11:35` - A user reports unexpected behaviour in #metadata
* `11:37` - SN is alerted to the possibility that data is missing from Elan 20220106
* `11:38` - SN confirms that data has been unpublished via NL
* `12:04` - SN on the scene and investigating
* `12:12` - SN opens #179 (https://github.com/COG-UK/dipi-group/issues/179)
* `12:24` - SN completes investigation and updates #179; noting the issue was with the `head` directory
* `12:29` - SN outlines options on #179
* `13:16` - SN confirms with Majora the number of affected sequences was 13,986
* `13:46` - SN updates the `published_date` of the affected sequences to occur one day in the future (2022-01-07)

### 2022-01-07
* `09:56` - SN confirms the number of expected sequences for Elan 20220107 includes both the rolled over sequences, and newly processed sequences
* `10:19` - Elan 20220107 is published, with no missing data
* `10:31` - SW confirms the integrity of the daily artifacts
* `11:27` - SN closes the incident


## Whys

* 171 sequences that were egregious malformed were processed by Elan
* ...because we underestimated the performance of `readfq` with bad data
* ...because we had never seen data this bad before

###
* 13,986 sequences that should have been published on 2022-01-05 were removed
* ...because an error was made when republishing the data set
* ...because there was no documentation for how to republish the data set
* ...because we have never had to do this before

## Technical cause

* Elan allowed malformed data to pass QC
* The `cog-publish` script allowed a user to republish the data set without the data processed that day

## Root cause

* Lenient assumptions on input data
* Lack of operational documentation

## Backlog check

* A consensus on what to do with non-IUPAC bases would have prevented `20220105_badfasta`
* Writing maintainer's documentation is a backlog task that may have prevented `20220105_unpublish` from happening.

## Recurrence

* Publishing sequences with non-IUPAC bases has happened before (as a feature), but publishing malformed FASTA has not.
* The data set has never needed republishing.

## Lessons learned

* Playbooks for disaster recovery, such as an emergency suppress-and-publish of the data set would be beneficial.

## Corrective actions

* [x] Immediately stop accepting sequences with non-IUPAC data (https://github.com/SamStudio8/elan-nextflow/commit/4828e44e50d4859d8612d06e03e69ca30c1b2dc9)
* [x] Ensure that the `cog-publish` script cannot be started if `head` is pointing to the same arifacts as `latest` (https://github.com/CLIMB-COVID/elan-nextflow/commit/c5e1a0651e5b9659335386060366d6a0ed9b5abc)
* [x] Add operational documentation for a data set republish

