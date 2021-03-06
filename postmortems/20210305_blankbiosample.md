| <img src="/assets/dipi.png" alt="DIPI Badge" width="150">      | Data Integrity and Pipeline Integration (DIPI) Working Group |
| -------------- | ----------------- |
| **Event**   | `20210305_blankbiosample` |
| **Systems**    | `Elan`, `Asklepian` |
| **Detected**   | 2021-03-05 23:27  |
| **Resolved**   | 2021-03-06 03:34  |
| **Team**       | SN                |
| **Related issues** | [#10](https://github.com/COG-UK/dipi-group/issues/10#issuecomment-791811466) |

## Summary

Elan 20210305 incorrectly processed 373 sequences related to incomplete `BiosampleArtifacts`.
One of these sequences caused the downstream analysis pipelines to terminate unexpectedly due to missing mandatory metadata.
The event was triggered by deployment of the new partial `update` API endpoint ([#10](https://github.com/COG-UK/dipi-group/issues/10)).
The change incorrectly allowed a PHA to update `BiosampleArtifact` records in Majora without providing all mandatory metadata fields.
The event was detected by `Asklepian` reporting an error to `#tael-stream`.
We have mitigated the issue by preventing the `update` endpoint from updating `BiosampleArtifact` records that do not have mandatory metadata and removing the offending sequence from the dataset.

## Leadup

On Friday 2021-03-05, the `d10-stomping` branch was merged into Majora production.
The branch provides a new `api.artifact.biosample.update` endpoint that permits partial updating of `BiosampleArtifact` objects in Majora.
The new update endpoint allows users to specify one or more fields to be updated for a given biosample in Majora.
Fields that are not included in the request are dropped from validation and are not updated in the target artifact -- solving the so-called "stomping" problem, where previously, missing fields were interpreted as `null` and the artifact was updated with `None`.

The change was tested with the Majora test suite as well as integration testing with the CGPS uploader.
However, the change did not adaquately consider a class of `BiosampleArtifact` in Majora: "empty biosamples".
Empty biosamples allow for select sites to register a biosample in advance, but not provide any metadata for it (with the intention to backfill it later).
This class of biosample are mostly used when large sites send samples out to smaller sequencing sites who need to be able to register the `LibraryArtifact` and `SequencingProcess`, which cannot be done if the `BiosampleArtifact` does not yet exist.

## Fault

After the feature was deployed, approximately 20,000 records were updated by a PHA throughout the afternoon of 2021-03-05; providing backfilled metadata and `root_biosample_source_id` for source-level deduplication.
These included 373 updates to "empty" `BiosampleArtifact` records that did not provide mandatory metadata.
Mandatory validation was skipped for 373 records as they were updated through the new endpoint.
The successful validation allowed the artifacts to be updated, and the `BiosampleArtifact.created.collection_location_adm0` and `BiosampleArtifact.created.submission_user` fields to be set (left unset for "blank" biosamples).
The 373 records were serialised and organised by `Ocarina` as part of the Elan 2021-03-05 pre-pipeline step but were not automatically filtered out as the `adm0` and `submission_user` fields were no longer blank.

## Impact

Elan processed the sequences with incomplete records, publishing 1 sequence associated with incomplete metadata (remaining sequences failed Elan QC, Basic QC or were ignored due to a separate issue).

## Detection

The issue was detected by `Asklepian`, which was guarded against the "impossible" situation of a sample missing both the `received_date` and `collection_date`.
`Asklepian` exited with an error code, automatically raising an `@channel` Slack message using the `Tael` mqtt wrapper around the `asklepian-phe` process to `#tael-stream` at 2021-03-05 23:27.

## Response

SN responded shortly after receiving the `@channel` alert as a push notification.

## Recovery

* Return `Asklepian` into service by removing the offending sequence from the Asklepian version of the day's data set and continuing from the point of termination
* Remove offending data from Majora
* Wait for offending sequence to be automatically pruned from the dataset the following day

## Timeline

### 2021-03-05 
* `10:40` - `d10-stomping` branch deployed to Majora production
* `16:01` - Elan begins
* `21:32` - Elan finishes and Asklepian begins
* `23:27` - Tael sends @channel alert to #tael-stream
    * SN receives push notification
    * SN checked the Asklepian error log, discovering a sequence without a received or collected date had been encountered
    * SN checked the sequence in `majora.metadata.tsv` and on the Majora web interface and confirmed the mandatory fields were missing
        * The new History table on the web interface allowed for quick confirmation that biosamples had been partially updated without the mandatory metadata
    * SN conducts further investigation to check how many samples are affected
 
### 2021-03-06
* `00:28` - [SN updates Github and Slack with a diagnosis](https://github.com/COG-UK/dipi-group/issues/10#issuecomment-791811466)
* `01:27` - [SN updates Github](https://github.com/COG-UK/dipi-group/issues/10#issuecomment-791829311), having discovered that 373 samples were affected by the issue, but is satisifed only a single sample is the cause of the error
* ~`01:30` - Confident this only affects a single sample; SN filters it from the day's `naive_msa.fasta` and `best_ref.ls`, SN manually raises Asklepian and waits to ensure completion. During this time SN failed to purchase a fancy GPU online.
* `01:54` - SN purges affected `PublishedArtifactGroup` from Majora and [updates Github](https://github.com/COG-UK/dipi-group/issues/10#issuecomment-791836726)
* `03:34` - SN manually sends a Tael message for `asklepian-phe` with `status=finished`, triggering downstream processing as normal
* `06:01` - Elan 20210306 runs at new start time
* `07:03` - Elan 20210306 completes, updating the `elan.consensus.fasta` to correctly remove the offending sequence

## Whys

* Asklepian terminated unexpectedly because there was a sequence with incomplete mandatory metadata
* ...because Elan processed sequences associated with incomplete metadata
* ...because Ocarina did not filter sequences associated with incomplete metadata
* ...because incomplete metadata is defined by missing entries for the `adm0` and `submitting_user` fields, which were not blank for these records
* ...because those fields were set when the records were partially updated by the new `api.artifact.biosample.update` endpoint
* ...because the endpoint allowed a user to partially update those records even though they were "blank" and needed mandatory metadata to be provided first
* ...because we did not consider blank records properly when developing this feature
* ...because we do not have a fully documented view of all the ways PHAs, sequencing sites and WSI expect to be able to update these records
* ...because we are not resourced to do this

## Technical cause

The `api.artifact.biosample.update` endpoint should not have allowed a user to update a record that is not completed.

## Root cause

We are not resourced appropriately to conduct user research on the ways metadata providers want to use Majora.

## Backlog check

No existing issues could have caught this.
User research to show how records in Majora are (or should be) updated by different sites would have been useful but is outside the scope of DIPI due to a lack of resource.

## Recurrence

This type of incident has not occurred before.

## Lessons learned

* A unit test that used the `addempty` endpoint may have highlighted this issue before deployment
* Ocarina should better guard Elan from processing samples with bad metadata
* The new History table on the artifact `detail_view` introduced by `d10-stomping` has improved debugging related to metadata updates
* Additional pressure was placed on responding to this issue as we had also moved the start of the next pipeline to 6am, leaving a short window to deploy a fix before running into processing the next Elan
* Need to be adaquately resourced to assemble a complete view of how updates to Majora are made when changing the `add` and `update` interface

## Corrective actions

* [x] Patch Majora such that the `api.artifact.biosample.update` endpoint refuses to update records that appear to be incomplete (https://github.com/SamStudio8/majora/commit/1fc9f7e309d9ac9f5d4895af882441969a7da471)
* [x] Write a Majora unit test that calls the `api.artifact.biosample.addempty` before trying to use `api.artifact.biosample.update` to ensure behaviour is maintained
* [x] Check the `collection_date`, `received_date` and `adm1` fields in `Ocarina`, rather than `adm0` and `submitting_user` to decide if a record is incomplete (https://github.com/SamStudio8/ocarina/commit/801d2ff614e4800af3c125d28ae5bd8622514afb)
* [x] Remove bad `PublishedArtifactGroup` from Majora to prevent public dissemination of sequences with incomplete metadata
* [ ] Suggest that someone at COG or PHA assemble a flow chart of desired update procedures to work from in future
* [ ] Suggest COG or PHA assign a business analyst or user researcher to work with us
