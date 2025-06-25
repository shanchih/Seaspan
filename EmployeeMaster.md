# IMOHCM27_HELM_HCM_EMPLOYEE_MASTER_OUTMAST_OUT_HCMEXTRACT
## Overview
This OIC integration is **scheduled** and uses the **HCM Extract Atom Feed** approach to retrieve new hire data and write it to an SFTP location.

## Integration Flow
| Step  | Description                                                                                                                                        |
| ----- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1 | **Schedule Trigger**: Initiated based on a schedule, captures `atomFeedLastRunDateTime` using a tracking variable.       |
| 2 | **Prepare AtomFeed Request**: Transformer builds the HCM Atom Feed request using the last run date.                       |
| 3 | **Invoke HCM Atom Feed**: Calls the `EmployeeNewHireFeed` operation via Oracle HCM Adapter.                           |
| 4 | **Route Based on Data**: Content-based router checks whether the response contains new hire data.                         |
| 5 | **Transform Data to File Format**: Transformer maps Atom Feed response into a structured file format.                     |
| 6 | **Stage File Write**: Writes transformed data to a temporary file using Stage File adapter.                               |
| 7 | **Transform for FTP Output**: Another transformer formats the staged file content for FTP upload.                         |
| 8 | **Write to SFTP**: File is uploaded to the /HELM/outbound/EmpMaster/ using FTP Adapter.                                  |
| 9 | **Invoke REST API (Optional)**: Optionally calls a REST API (`getEmpDetalisRest`) using Rest Invoke /hcmRestApi/resources/11.13.18.05/workers/.  <font color='red'>Review !</font> **REVIEW NEEDED**
|    | **End**: Integration ends with a `stop` processor.   Needed. Not clear the purpose of this step</font>
|    | **End**:                                                                                      |

## Additional Notes

- **`atomFeedLastRunDateTime`**:A schedule parameter used to track the last successful feed poll. Ensures only delta/new records are fetched.
- **Adapters Used**:

  - **HCM Adapter** (`getNewHireFeed`) for Atom Feed extraction
  - **Stage File Adapter** for temporary file creation
  - **FTP Adapter** for final file delivery
  - **REST Adapter** (`getEmpDetalisRest`) for optional data enrichment
- **Transformers**:
  - Three transformers are used for:
    - Preparing AtomFeed request
    - Converting AtomFeed response to file format
    - Reformatting file content before FTP upload
- **Message Tracking**:A global `messageTracker` captures key metadata like `startTime`.
- **Error Handling**:
  Defined fault handlers (`APIInvocationError`) for both HCM Adapter and REST Adapter invocations.

# IMOHCM27_HELM_HCM_EMPLOYEE_MASTER_OUT
## Overview
This OIC integration is designed to extract employee data (both new hires and updates) from Oracle HCM Cloud using Atom Feeds and deliver it to an SFTP location. The integration follows a scheduled approach with delta tracking capabilities.

Step

Processor / Application

Description

1️⃣

`scheduleReceive` + `messageTracker`

Triggered by schedule. `startTime` and `atomFeedLastRunDateTime` captured using tracking variables.

2️⃣

Transformer (`processor_36`)

Constructs the Atom Feed request using the last run datetime.

3️⃣

HCM Adapter `getNewHireFeed`

Calls **`EmployeeNewHireFeed`** operation to fetch new hire data.

4️⃣

Router (`processor_45`)

Routes data based on whether feed has new records:• If **new hires present**, proceed to file creation.• If not, alternate path updates metadata (uses `getUpdateWorker`).

5️⃣

Transformer (`processor_70`)

Maps Atom Feed response to file structure.

6️⃣

Stage File Write (`processor_56`)

Writes structured data to a temp file.

7️⃣

Transformer (`processor_93`)

Prepares data for FTP output formatting.

8️⃣

FTP Adapter `writeFileToFTP`

Uploads file to `/HELM/outbound/EmpMaster/`.

9️⃣

REST Adapter `getEmpDetalisRest`

Optional enrichment call to `/hcmRestApi/resources/11.13.18.05/workers/`. (**Review Needed**)

🔚

Stop (`stop`)

Terminates the integration process.
<!--stackedit_data:
eyJoaXN0b3J5IjpbMjA4ODg3MDk3MywtMTE2MzAxNzEzNywzNj
AwODM0NDIsLTEwNzgyNjA3MDUsMTQxNTM0ODgxNSwtMTExNDg3
NjY1MSwtODI3OTQ1Njg2LC02MjIxNDQ3MTFdfQ==
-->