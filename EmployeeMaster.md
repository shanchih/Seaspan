# IMOHCM27_HELM_HCM_EMPLOYEE_MASTER_OUTMAST_OUT_HCMEXTRACT
## Overview
This OIC integration is **scheduled** and uses the **HCM Extract Atom Feed** approach to retrieve new hire data and write it to an SFTP location.

## Integration Flow
| Step  | Description                                                                                                                                        |
| ----- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1 | **Schedule Trigger**: Initiated based on a schedule, captures `atomFeedLastRunDateTime` using a tracking variable.       |
| 2 | **Prepare Atom Feed Request**: Transformer - build HCM Atom Feed request using the last run date time.                       |
| 3 | **Invoke HCM Atom Feed**: Calls the `EmployeeNewHireFeed` operation via Oracle HCM Adapter                    |
| 4 | **Router**: checks whether the response contains new hire data. (EmployeeNewHireFeed_Update > 0)                        |
| 5 | **Transform Data to File Format**: Transformer-maps Atom Feed response into a structured file format.                     |
| 6 | **Stage File Write**: Writes transformed data to a temporary file using Stage File adapter.                               |
| 7 | **Transform for FTP Output**: Another transformer formats the staged file content for FTP upload.                         |
| 8 | **Write to SFTP**: File is uploaded to the /HELM/outbound/EmpMaster/ using FTP Adapter.                                  |
| 9 | **🔴Invoke REST API **:calls a REST API (`getEmpDetalisRest`) /hcmRestApi/resources/11.13.18.05/workers/. ❗What is the purpose of this step?
| 10| **Assign**: atomFeedLastRunDateTime = startTime
                                                                   


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
- **Error Handling**:  🔴 No fault handler to capture `APIInvocationError` for both HCM Adapter and Rest Adppater ations.

# IMOHCM27_HELM_HCM_EMPLOYEE_MASTER_OUT
## Overview
This integration is designed to extract employee data (both new hires and updates) from Oracle HCM Cloud using Atom Feeds and deliver it to an SFTP location. The integration follows a scheduled approach with delta tracking capabilities. 🔴 This integration seems a duplicated to the previous integration 

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
    A[Schedule Trigger] --> B[st]
BGet Integration Metadata]
    B --> C[Transform Schedule Data]
    C --> CD[Invoke HCM AdpaterCloud\nEmployeeNewHire --> egration Metadata]D{Router}
```


```mermaid 
flowchart TD
A[Schedule Trigger]Feed]
    D --> E{Content-Based Router}
    
    E --> |New Hire Path| F[Transform New Hire Data]
    F --> G[Stage File]
    G --> BH[Transformer: Build Reque --> [Transform]
     --> [Invoke HCM \nEmployee New Hire]
C --> Get Integration Metadata]D{Router}
```

```mermaid
flowchart TD
    A[Schedule Trigger] --> B[Get Integration Metadata]
    B --> C[Transform Schedule Data]
    C --> D[Invoke HCM Cloud\nEmployeeNewHireFeed]
    D --> E{Content-Based Router}
    
    E --> |New Hire Path| F[Transform New Hire Data]
    F --> G[Stage File]
    G --> H[Transform File Data]
    H --> I[Write File to FTP]
    
    E --> |Update Path| J[Transform Update Data]
    J --> K[Invoke HCM Cloud\nEmployeeUpdateFeed]
    
    C --> L[Transform for REST]
    L --> M[Invoke REST\nGet Employee Details]
    
    I --> N[Stop]
    K --> N
    M --> N```
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTU1NzA5MDAxMywtMTE4NzQ3NzM4MCw3MT
YzNTYzMTYsOTY3NjAzNzUwLDI3ODg0MTk5OCwtMjQ0ODYyNDY0
LC0xMDk1ODQzODcyLDE5NzkxMDU1NDEsMjExMzUxOTcxLDEzMz
AxNjIyNTUsLTIyMTYyNDQ0OSwxODMwNDE1NzA5LC0yMTMyNTAz
NjY5LDM0NDA3NTE2OSwtMjA0OTY5Mjg0OSwxNDE0OTk5ODA3LD
UyODExMTg4OSwxNzgyODM5NTEyLDEyNjEwNTAxMDQsMTMyNTQ3
OTkwXX0=
-->
