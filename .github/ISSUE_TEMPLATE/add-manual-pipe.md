---
name: Manual pipeline protocol
about: Open an optional tracking issue for a manually run pipeline
title: "[manualpipe] "
labels: control
assignees: 
---

**Brief description**

[Provide a sentence of background]

**Pipeline exit code:** [Exit code]

***
**Executed by:** [Name or initials] [Site code]

* [ ] I am sure this pipeline is not already running (**Only relevant for non-flocked pipelines**)

***
* [ ] Engage pipeline via mqtt **from a clean dedicated mqtt environment**
* [ ] Disable NXF cache destruction (only applicable for ENA BAM pipe)
* [ ] Check output if necessary
* [ ] Check mqtt message is sent confirming completion, or send one if required (check in #tael-steam)
* [ ] Re-enable NXF cache destruction (if disabled)

***
* [ ] Set appropriate issue flags and assignee

* [ ] Manual intervention complete
* "Normal" service disrupted? **YES/NO**
* If yes, describe impact:
