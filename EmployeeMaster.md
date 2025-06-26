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
  - Otherwise:  ❗What is the purpose of this path?   
    - Calls HCM Cloud Adpater `getUPdateWorker` ❗What is the purpose of this path?
6. Rest call to HCM using `getEmpDetailsRest` ❗What is the purpose?
7. Assign `atomFeedLastRunDateTime` = starttime for next scheduled execution

# IMOHCM27_EMPLOYEE_EMERGENCY_CONTACTS_HELM
## Overview
This integration is designed to manage employee emergency contacts by extracting data from Oracle HCM Cloud, processing it, and synchronizing it with a HELM system (likely a third-party HR. It includes error handling, logging, and notifications for success or failure.
- Extracts employee emergency contact data from Oracle HCM Cloud.
- Processes the data and sends it to HELM via REST APIs.
- Includes retry logic for failed operations.
- Logs activities and errors, and sends notifications.
- Supports both creating and updating emergency contacts in HELM.

##  Integration Flow

1. **Initialization**
  - **Schedule Trigger**: The integration starts with a scheduled trigger with startTime as the tracking variable.
  - **Metadata Setup**: Initializes phase variables like startTime, g_RiceId, g_FlowName, g_FromEmail, g_ToEmail, etc.

2. **Extract Data from HCM**
- **Submit Extract Request**:
  - Transforms the request payload
  - Calls the HCM SOAP service to submit an extract request
- **Poll for Extract Status**:
  - Uses a while loop to periodically check the status of the extract.
  - Waits and checks the status via another SOAP call.
- **Download Extract**:
  - Once the extract is ready, downloads the file from Oracle HCM Cloud.
3. **Process Extracted Data**
- **Stage File**: Reads the downloaded file.
- **Log Header**: Writes a log header for tracking.
- **For Each Record**: Processes each record in the extracted file.
  - **Get HELM Employee Data**:
    - Transforms and sends a REST request to HELM to fetch employee details.
    - Routes based on the response
  - **Get HELM Contact Data**:
    - For valid employees, fetches existing contact data from HELM.
    - Routes based on whether the contact exists.
      - **Create Contact**: If the contact doesn't exist, sends a REST request to create it.
      - **Update Contact**: If the contact exists, sends a REST request to update it.
  - **Error Handling**:
    - Catches errors and updates error counts
    - Logs errors and updates status variables.
4. **Logging and Notifications**
- **Write Log File**: Logs the processing results.
- **Read Log File**: Reads the log file for further processing.
- **Write FTP Log**: Uploads the log file to an FTP server (application_1866).
- **Send Notifications**:
  - **Success**: Sends a success notification if all records are processed/
  - **Error**: Sends an error notification if any failures occur.

5. **Final Steps**
- **Stop**: Ends the integration.
- **Global Error Handling**: Catches any unhandled errors and sends an error notification

nferred HELM API Structure
Endpoint	Method	Purpose	Used When
/employees/{id}	GET	Fetch employee details	Validating employee in HELM
/employees/{id}/contacts	GET	Fetch employee’s contacts	Check if contact exists
/contacts	POST	Create new emergency contact	No existing contact found
/contacts/{id}	PUT/PATCH	Update existing emergency contact	Contact already exists in HELM

## HELM REST API Structure (Inferred from Integration)

| Endpoint                          | Method | Purpose                               | Trigger Condition                |
|-----------------------------------|--------|---------------------------------------|----------------------------------|
| `/employees/{employeeId}`         | GET    | Get employee details                  | Initial employee lookup          |
| `/employees/search`               | POST   | Search for employees (alternate)      | If GET by ID not available       |
| `/employees/{employeeId}/contacts`| GET    | Get employee's emergency contacts     | After employee validation        |
| `/contacts`                       | POST   | Create new emergency contact          | When no existing contact found   |
| `/contacts/{contactId}`           | PUT    | Fully update emergency contact        | When contact exists              |
| `/contacts/{contactId}`           | PATCH  | Partially update emergency contact    | When contact exists (alternate)  |

**Key Notes:**
1. All endpoints likely require authentication (API key/OAuth2)
2. Error responses use `APIInvocationError` format
3. Integration performs existence check before create/update
4. XSLT transformations map between HCM and HELM data formats

**Sample Request Flow:**
1. `GET /employees/12345` → Validate employee
2. `GET /employees/12345/contacts` → Check for existing contact
3. `POST /contacts` or `PUT /contacts/67890` → Create/Update contact

## Initialization Phase Variables

| Variable Name            | Type    | Purpose |
|--------------------------|---------|---------|
| `startTime`              | Tracking | Timestamp when integration started |
| `g_RiceId`               | String  | Unique identifier for the integration process |
| `g_FlowName`             | String  | Name of the integration flow |
| `g_FromEmail`            | String  | Sender email for notifications |
| `g_ToEmail`              | String  | Recipient email for notifications |
| `g_Desc`                 | String  | Description of the integration |
| `g_Instance`             | String  | Instance identifier (environment-specific) |
| `g_EffectiveDate`        | String  | Date for data processing |
| `g_IntegrationName`      | String  | Name of the integration |
| `g_IntegrationTimeStamp` | String  | Timestamp for integration execution |
| `g_FlowInstanceStatus`   | String  | Status of the flow instance (e.g., "IN_PROGRESS") |
| `g_Process_ID`           | String  | Process ID for tracking |
| `g_OutboundDirectory`    | String  | File path for output logs/extracts |
| `g_FileName`             | String  | Name of the extracted file |
| `g_SuccessCount`         | String  | Counter for successful operations |
| `g_ErrorCount`           | String  | Counter for failed operations |
| `g_TotalCount`           | String  | Total records to process |
| `g_SendDataToHELM`       | String  | Flag to determine if data should be sent to HELM |
| `g_Status`               | String  | Current status of the integration |
| `g_Message`              | String  | Status or error message |
| `g_logFileName`          | String  | Name of the log file |
| `sch_documentID`         | String  | Document ID for scheduling |
