---
name: Add a new sequencing platform
about: Request to add a sequencing platform to Majora and friends
title: "[platform] "
labels: metadata
assignees: SamStudio8
---

**Brief description**

[Provide a sentence of background]

**Platform**

Provide the platform name and suggest an option for the platform validator.

**QC**

Provide the proposed QC rules below for Basic and High QC. See the [QC docs](https://docs.covid19.climb.ac.uk/how-data).

**Dehumanisation**

Provide the suggested options for human reference mapping. See the [`dehumanise_bam` pipeline step](https://github.com/SamStudio8/elan-nextflow/blob/master/bam/dehuman.nf).


**ENA submissions**

Ensure the platform is formally listed [here](https://github.com/enasequence/schema/blob/master/src/main/resources/uk/ac/ebi/ena/sra/schema/SRA.common.xsd).

***
**Proposed by:** [Name/Initials] [Site code]

* [ ] I have read the documentation for the existing validators and there is no suitable entry

***
***

**FOR DEVELOPER USE**

* [ ] Add field validation to Majora forms
* [ ] Add handling to Majora QC subsystem
* [ ] Update pyENA
* [ ] Add field to API docs
* [ ] Push to Majora prod
* [ ] Update validator spreadsheet (if applicable)
