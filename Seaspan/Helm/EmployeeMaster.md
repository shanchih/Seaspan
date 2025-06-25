# IMOHCM27_HELM_HCM_EMPLOYEE_MASTER_OUTMAST_OUT_HCMEXTRACT
## Overview
This OIC integration is **scheduled** and uses the **HCM Extract Atom Feed** approach to retrieve new hire data and write it to an SFTP location.

## Integration Flow
| Step  | Description                                                                                                                                        |
| ----- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1 | **Schedule Trigger**: Initiated based on a schedule, captures `atomFeedLastRunDateTime` using a tracking variable.       |
| 2 | **rarmed Request**: the HCM Atom Feed request using the last run date time.                       |
| 3 | **e HCM Atom Feed**: Calls the `EmployeeNewHireFeed` operation via                     |
| 4 | **Content-based router**: checks whether the response contains new hire data. (EmployeeNewHireFeed_Update > 0)                        |
| 5 | **Transform Data to File Format**: Transformer maps Atom Feed response into a structured file format.                     |
| 6 | **Stage File Write**: Writes transformed data to a temporary file using Stage File adapter.                               |
| 7 | **Transform for FTP Output**: Another transformer formats the staged file content for FTP upload.                         |
| 8 | **Write to SFTP**: File is uploaded to the /HELM/outbound/EmpMaster/ using FTP Adapter.                                  |
| 9 | **Invoke REST API (Optional)**: Optionally calls a REST API (`getEmpDetalisRest`) using Rest Invoke /hcmRestApi/resources/11.13.18.05/workers/.  <font color='red'>Review !</font> **REVIEW NEEDED**
|    | **End**: Integration ends with a `stop` processor.   Needed. Not clear the purpose of this step</font>
|    | **End**:                                                                                      |

```mermaid 
sequenceDiagram
    participant Scheduler as Scheduler (processor_8)
    participant MessageTracker as Message Tracker (processor_1)
    participant HCMCloud as HCM Cloud (getNewHireFeed)
    participant Router as Content Router (processor_45)
    participant StageFile as Stage File (processor_56)
    participant SFTP as SFTP Adapter (writeFileToFTP)
    participant Assignment as Assignment (processor_101)
    participant HCMRest as HCM REST (getEmpDetalisRest)

    Note over Scheduler: Scheduled Trigger (INT-29 HCM Extract)
    Scheduler->>MessageTracker: Schedule Received (startTime tracking)
    activate MessageTracker
    MessageTracker-->>Scheduler: Tracking variables captured
    deactivate MessageTracker

    Scheduler->>HCMCloud: EmployeeNewHireFeed Request
    activate HCMCloud
    HCMCloud-->>Scheduler: EmployeeNewHireFeed Response
    deactivate HCMCloud

    Scheduler->>Router: Route message
    activate Router
    Router->>StageFile: Write Records Request
    activate StageFile
    StageFile-->>Router: Write Response
    deactivate StageFile
    
    Router->>SFTP: WriteFile Request
    activate SFTP
    SFTP-->>Router: WriteFile Response
    deactivate SFTP
    Router-->>Scheduler: Routing complete
    deactivate Router

    Scheduler->>Assignment: Execute variable assignment
    activate Assignment
    Assignment-->>Scheduler: Assignment complete
    deactivate Assignment

    Scheduler->>HCMRest: execute Request (Employee Details)
    activate HCMRest
    HCMRest-->>Scheduler: execute Response
    deactivate HCMRest

    Note over Scheduler: Integration Completed (100%)
```


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

##  Integration Flow

## 🧭 Integration Steps

| Step  | Flow Description                              |
| ----- | -------------------------------------------------------------------------------------------------- |
| 1 | **Schedule Trigger** — Triggered on a schedule. Captures `atomFeedLastRunDateTime` using message tracking.  |
| 2 | **Transformer: Build Request** — Constructs the Atom Feed request using the last run datetime.           |
|  | **HCM Adapter: Get Atom Feed** — Invokes `EmployeeNewHireFeed` to fetch new hires from Oracle HCM.      |
| 4 | **Content-Based Router** — Evaluates whether response contains new data and routes accordingly:                                       |
|       |  •**New HireData Exists**:  (EmployeeNewHireFeed_Update > 0)                                                                                 |
|       | &nbsp;&nbsp;&nbsp;&nbsp;– **Transformer:** Maps Atom Feed response to flat file format.         |
|       | &nbsp;&nbsp;&nbsp;&nbsp;– **Stage File** Writes transformed data to a temp file using Stage File Adapter.  |
|       | &nbsp;&nbsp;&nbsp;&nbsp;– **Transformer:** Formats the staged content for FTP upload.           |
|       | &nbsp;&nbsp;&nbsp;&nbsp;– **FTP Adapter:** Uploads the file to SFTP at `/HELM/outbound/EmpMaster/`.                  |
|       | •**Otherwise**:                 
|
|       | &nbsp;&nbsp;&nbsp;&nbsp;– **Transformer: Prepare Request** — Constructs request to `getUpdateWorker` endpoint. |
|       | &nbsp;&nbsp;&nbsp;&nbsp;– **HCM Adapter: getUpdateWorker** — Sends request to update worker metadata.                  |  **REST Adapter (Optional)** — Optionally calls REST API `getEmpDetalisRest` to fetch more worker info. (**Review if needed**) |
| 🔚    |  **Stop** — Ends the integration.                                                            |
<!--stackedit_data:
eyJoaXN0b3J5IjpbMzQ0MDc1MTY5LC0yMDQ5NjkyODQ5LDEyNj
EwNTAxMDQsMTMyNTQ3OTkwLC0xMDg5NjQ1NTgzLDEyNTUwNjQx
MjQsLTExNjMwMTcxMzcsMzYwMDgzNDQyLC0xMDc4MjYwNzA1LC
0xMTE0ODc2NjUxLC02MjIxNDQ3MTFdfQ==
-->