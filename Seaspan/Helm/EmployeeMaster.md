# IMOH_HELM_HCM_EMPL_MAST_OUT_HCME
This OIC integration is **scheduled** and uses the **HCM Extract Atom Feed** approach to retrieve new hire data and write it to an SFTP location.

| Step  | Description                                                                                                                                        |
| ----- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1 | **Schedule Trigger**: Initiated based on a schedule, captures `atomFeedLastRunDateTime` using a tracking variable.       |
| 2 | **Prepare AtomFeed Request**: Transformer (`processor_36`) builds the HCM Atom Feed request using the last run date.                       |
| 3 | **Invoke HCM Atom Feed**: Calls the `EmployeeNewHireFeed` operation via Oracle HCM Adapter (`application_24`).                           |
| 4 | **Route Based on Data**: Content-based router (`processor_45`) checks whether the response contains new hire data.                         |
| 5 | **Transform Data to File Format**: Transformer (`processor_70`) maps Atom Feed response into a structured file format.                     |
| 6 | **Stage File Write**: Writes transformed data to a temporary file using Stage File adapter (`processor_56`).                               |
| 7 | **Transform for FTP Output**: Another transformer (`processor_93`) formats the staged file content for FTP upload.                         |
| 8 | **Write to SFTP**: File is uploaded to the designated SFTP location using FTP Adapter (`application_83`).                                  |
| 9 | **Invoke REST API (Optional)**: Optionally calls a REST API (`getEmpDetalisRest`) using the Oracle HCM REST Adapter (`application_128`). |
|     | **End**: Integration ends with a `stop` processor.                                                                                         |

---

# 🔍 Special Notes

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
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTIxMjIwMjA4MiwtNjIyMTQ0NzExXX0=
-->