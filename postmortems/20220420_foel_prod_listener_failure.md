| <img src="/assets/dipi.png" alt="DIPI Badge" width="150"> | Data Integrity and Pipeline Integration (DIPI) Working Group |
| --------------------------------------------------------- | ------------------------------------------------------------ |
| **Event**                                                 | `20220420_foel_prod_listener_failure`                        |
| **Systems**                                               | `Foel`                                                       |
| **Detected**                                              | `2022-04-28 10:34`                                           |
| **Resolved**                                              | `2022-04-29 12:01`                                           |
| **Team**                                                  | `SW`, `NL`, `BM`                                             |

## Summary

A site-wide DNS glitch resulted in the Foel production SQS message listener failing since it could not connect to AWS endpoint at `2022-04-20 19:48`. This listener has no way to re-raise itself or indicate that it has failed which lead to this failure being undetected for eight days until `2022-04-28 10:34` when `BM` sent an email to `SW` and `NL`. This email indicated that no genomes had been recorded as received via the AWS ingest since `2022-04-21` despite sequences having been submitted. In response to this `SW` determined that the `foel_prod` systemd service had failed and restarted it successfully.

## Leadup

* Foel had been successfully ingesting genomes into CLIMB-COVID via AWS since its introduction in late 2021.
* As part of a wider migration to systemd services (Operation Antelope) `foel_prod` was successfully migrated to the newly made services VM from the CLIMB-COVID login node on `2022-03-08`.
* While migrating `foel_prod` to a systemd service no failure mitigations were implemented such as an automatic restart clause or an automated alert on the failure of the service.

## Fault

### `20220420_foel_prod_listener_failure`
The `foel_prod` listener is responsible for polling and consuming the AWS SQS message queue containing genomes and associated metadata provided by UKHSA via private providers. It achives this by polling the AWS SQS endpoint at a set regular interval, if it discovers new messages (json payloads) it then consumes these and automatically draws the files referred to in the messages onto CLIMB-COVID for further processing by the main Foel pipeline. 

If the listener is unable to connect to the correct AWS endpoint URL `https://eu-west-2.queue.amazonaws.com/` the listener will fail leading to the systemd service responsible for running it to also fail. At `2022-04-20 19:48` this occured due to a site-wide DNS glitch with the error message:

```
Apr 20 19:48:39 climbcovid-services.novalocal run_foel_prod.sh[7302]: botocore.exceptions.EndpointConnectionError: Could not connect to the endpoint URL: "https://eu-west-2.queue.amazonaws.com/"
```

## Impact

* 861 SQS messages relating to 287 genomes submitted by private providers were delayed in their processing.
* 423 SQS messages relating to 141 genomes expired due to their age on `2022-04-25` meaning they were not ingested.

## Detection

* `20220420_foel_prod_listener_failure` -- `BM` noticed that splunk had not reported any successfully processed genomes since the `2022-04-21` and that oldest message age was steadily increasing.

## Response

* `20220420_foel_prod_listener_failure` -- SW restarted `foel_prod` listener at `2022-04-28 10:39`

## Recovery

### `20220420_foel_prod_listener_failure`
* Restart prod listener
* Ensure SQS queue is being consumed and processed correctly for the delayed messages
* Re-submit the SQS messages for the 141 genomes affected by message expiration so those genomes may be ingested (**ongoing**) 
* Introduce monitoring of SQS queue attributes to prevent such a delayed response for recurring
* Add `foel_prod` service failure notifications ensuring that a failure is not detected quickly in future.

## Timeline

### 2022-04-20
* `19:48` - `foel_prod.service` fails due to a DNS glitch preventing the service from accessing the `https://eu-west-2.queue.amazonaws.com/` AWS endpoint.

### 2022-04-21
* `throughout` - `x` genomes submitted to the AWS ingest.

### 2022-04-25
* `throughout` - messages for the `x` genomes submitted on the `2022-04-21` expire.

### 2022-04-28
* `10:34` - BM sends email to SW and NL observing that SQS messages are aging and some have expired.
* `10:38` - SW checks the status of `foel_prod` systemd service and observes that it has failed.
* `10:39` - SW restarts the `foel_prod` systemd service.
* `10:40` - SW confirms that the SQS messages are being properly consumed and processed by foel.
* `10:45` - BM confirms that SQS messages are being consumed via Splunk.
* `12:45` - SW implements regular monitoring of the SQS queue to ensure that the production queue is either empty or reducing in size every 6 hours.

### 2022-04-29
* `12:01` - SW implements `foel_prod` failure notification to be triggered by the failure of the systemd service.

## Whys

* The failure of the `foel_prod` listener systemd service was not noticed by administrators for 8 days. 

## Technical cause

* A DNS glitch prevented `foel_prod` from connecting to the AWS endpoint leading to the listener failing.
  
## Root cause

* Lack of effective monitoring of the SQS queue.
* Lack of effective `foel_prod` service failure mitigation.

## Recurrence

* This is the first occurence of such an issue.

## Lessons learned

* Better alerts for systemd service failures is necessary, can be applied to several systemd services, consider automatic restarts as well as alert messages.

## Corrective actions

* [x] Implement automated SQS queue monitoring so that SQS queue issues can be detected quickly.
* [x] Add a systemd service triggered on the failure of `foel_prod` to alert administrators that the service has failed.
* [ ] Re-submit the SQS messages for the 141 genomes affected by message expiration