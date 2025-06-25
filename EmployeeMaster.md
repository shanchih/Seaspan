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
| 9 | **Invoke REST API (Optional)**: Optionally calls a REST API (`getEmpDetalisRest`) using Rest Invoke /hcmRestApi/resources/11.13.18.05/workers/.  <font color='red'>Review Needed!</font>
| 10   | **Assign**: atomFeedLastRunDateTime = startTime
|    | **End**:                                                                         


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
- **Message Tracking**:A global `messageTracker` captures  `startTime`.
- **Error Handling**:  <font color='red'>NoReview Needed!</  Defif fault handler to capture `APIInvocationError` for both HCM Adapter and Rest Adppater ations.</font> 

# IMOHCM27_HELM_HCM_EMPLOYEE_MASTER_OUT
## Overview
This OIC integration is designed to extract employee data (both new hires and updates) from Oracle HCM Cloud using Atom Feeds and deliver it to an SFTP location. The integration follows a scheduled approach with delta tracking capabilities. <font color='red'>This integration seems a replicated to the previous integrtion 

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

```mermaid
flowchart TD
    A[Schedule Trigger] --> B[Transformer:Build request]
    B --> C[Invoke HCM Adapter\nEmployeeNewHireFeed]
    C --> D{Content-Based Router}
    D --> E{Content-Based Router}
    
    E --> |New Hire Path| F[Transform New Hire Data]
    F --> G[Stage File]
    G --> H[Transform File Data]
    H --> I[Write File to FTP]
    
    E --> |Update Path| J[Transform Update Data]
    J --> K[Invoke HCM Cloud\nEmployeeUpdateFeed]
    
    
    I --> N[Stop]
    K --> N


```

```mermaid
sequenceDiagram
    participant Scheduler
    participant Integration
    participant HCM_NewHire
    participant HCM_Update
    participant FTP
    participant REST
    
    Scheduler->>Integration: Trigger (scheduleReceive)
    Integration->>Integration: Get Metadata
    Integration->>Integration: Transform Schedule Data
    Integration->>HCM_NewHire: EmployeeNewHireFeed
    
    alt New Hire Path
        HCM_NewHire-->>Integration: New Hire Response
        Integration->>Integration: Transform New Hire Data
        Integration->>Integration: Stage File
        Integration->>FTP: Write File
        FTP-->>Integration: Write Response
    else Update Path
        HCM_NewHire-->>Integration: Update Response
        Integration->>Integration: Transform Update Data
        Integration->>HCM_Update: EmployeeUpdateFeed
        HCM_Update-->>Integration: Update Response
    end
    
    par REST Call
        Integration->>Integration: Transform for REST
        Integration->>REST: Get Employee Details
        REST-->>Integration: Employee Data
    end
    
    Integration->>Integration: Stop

```
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTg3NjAzNzc1NCwyNzg4NDE5OTgsLTI0ND
g2MjQ2NCwtMTA5NTg0Mzg3MiwxOTc5MTA1NTQxLDIxMTM1MTk3
MSwxMzMwMTYyMjU1LC0yMjE2MjQ0NDksMTgzMDQxNTcwOSwtMj
EzMjUwMzY2OSwzNDQwNzUxNjksLTIwNDk2OTI4NDksMTQxNDk5
OTgwNyw1MjgxMTE4ODksMTc4MjgzOTUxMiwxMjYxMDUwMTA0LD
EzMjU0Nzk5MCwxODE1NjE2MTQ5LC0xMDg5NjQ1NTgzLDg2NzUz
NDk4Nl19
-->