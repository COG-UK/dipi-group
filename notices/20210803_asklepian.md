| <img src="/assets/dipi.png" alt="DIPI Badge" width="150">      | Data Integrity and Pipeline Integration (DIPI) Working Group</br>Change Advisory Notice |
| -------------- | ----------------- |
| **Event**      | Asklepian MSA engine  |
| **Systems**    | `asklepian-phe`  |
| **Data**    | `asklepian` downstream data (MSA, genome table, variant table)  |
| **Implemented**| Awaiting approval |
| **Related issues** | [#117](https://github.com/COG-UK/dipi-group/issues/117) |

## Summary

To reduce the processing time taken by Asklepian, [DIPI#117](https://github.com/COG-UK/dipi-group/issues/117) will swap the MSA generation subroutine from `datafunk` to `gofasta`.
This change affects the entries in the MSA for 14 existing sequences in the COG-UK data set (compared to `datafunk`) which will additionally trigger changes in the Asklepian genome table and variant table (and anything derived from them).

## Analysis

Alignment of these 14 sequences with `minimap2` during the MSA generation process causes `datafunk` to report an "ambiguous overlapping alignment".
This appears to be an edge case in `datafunk` which is handled more appropriately in `gofasta`, yielding a sequence with fewer masked positions.

The MSA for the following COG-IDs will be updated by this change:

* LOND-12F4E00
* LOND-132BA74
* NIRE-23CEC3
* NORT-1BD7620
* NORW-13E31A0
* NORW-13EA6C8
* NORW-225953
* NORW-F841E
* OXON-F5EE73
* PHWC-4A914E
* PHWC-4C412F
* PHWC-PYKGNE
* QEUH-1237214
* QEUH-1237D22

A brief check determined that the updated MSA will not affect variant calling for any of the sequences.

## Recommendation

We do not believe user action is required to mitigate this change, but downstream users should be aware of the changes to these records.
