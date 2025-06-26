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

1. Starts with a scheduled trigger : Tracking variables `startTime`
2. Gets integration metadata
3. Transforms request for HCM Cloud : Prepare HCM request
4. Calls HCM Adpater to get new hire feed
5. Routes the response:
  - If new hires : Send to FTP
    - Transforms data
    - Stages files
    - Transforms stage output
    - Writes to FTP
  - Otherwise:
    - Transforms data
    - Calls HCM Cloud Adpater `getUPdateWorker`
6. Rest call to HCM using `getEmpDetailsRest` 
7. Assign `atomFeedLastRunDateTime` = starttime for next scheduled execution



