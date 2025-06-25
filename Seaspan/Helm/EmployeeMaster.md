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

## Integration Flow

| Step  | FlowDescription                                                                                                                                                                       |
| ----- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1️⃣ | **Schedule Trigger** — Triggered on a schedule. Captures `atomFeedLastRunDateTime` using message tracking.                                                                    |
| 2️⃣ | **Transformer: Build Request** — Constructs the Atom Feed request with the last run datetime using XSLT (`processor_36`).                                                     |
| 3️⃣ | **HCM Adapter: Get Atom Feed** — Invokes `EmployeeNewHireFeed` to fetch new hires from Oracle HCM.                                                                            |
| 4️⃣ | **Content-Based Router** — Evaluates whether response contains new data:`<br>`– If **data exists**: go to file flow `<br>`– If **no data**: go to update path |
| 5️⃣ | **Transformer: Format for File** — Maps Atom Feed response to flat file format (`processor_70`).                                                                              |
| 6️⃣ | **Stage File Write** — Writes transformed data to a temp file using Stage File Adapter.                                                                                         |
| 7️⃣ | **Transformer: Prepare for FTP** — Formats the staged content for FTP upload (`processor_93`).                                                                                |
| 8️⃣ | **FTP Adapter: Upload File** — Uploads the file to SFTP at `/HELM/outbound/EmpMaster/`.                                                                                       |
| 9️⃣ | **REST Adapter (Optional)** — Optionally calls REST API `getEmpDetalisRest` to fetch more worker info. (**Review if needed**)                                           |
| 🔚    | **Stop** — Ends the integration.                                                                                                                                                |

<!--stackedit_data:
eyJoaXN0b3J5IjpbMjAwODc4MzYwMCwtMTE2MzAxNzEzNywzNj
AwODM0NDIsLTEwNzgyNjA3MDUsLTExMTQ4NzY2NTEsLTYyMjE0
NDcxMV19
-->