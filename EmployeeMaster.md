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
| 9 | **🔴Invoke REST API **:calls a REST API (`getEmpDetalisRest`) /hcmRestApi/resources/11.13.18.05/workers/. ❗`Alex-What is the purpose of this step?`
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
- **Error Handling**:  🔴 `Alex-No fault handler to capture `APIInvocationError` for both HCM Adapter and Rest Adppater ations.`

# IMOHCM27_HELM_HCM_EMPLOYEE_MASTER_OUT
## Overview
This integration is designed to extract employee data (both new hires and updates) from Oracle HCM Cloud using Atom Feeds and deliver it to an SFTP location. The integration follows a scheduled approach with delta tracking capabilities. 🔴 `Alex-This integration seems a duplicated to the previous integration` 

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
  - Otherwise:  ❗Alex-`What is the purpose of this path?`   
    - Calls HCM Cloud Adpater `getUPdateWorker` ❗Alex-`What is the purpose of this path?`
6. Rest call to HCM using `getEmpDetailsRest` ❗Alex-`What is the purpose?`
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
  - Uses a while loop to periodically check the status of the extract. (g_FlowInstanceStatus = 'READY' or 'WAIT' or 'PAUSED' or 'RUNNING' ❗`Do we need READY?` )
  - Waits and checks the status via Check Extract Status SOAP call.
- **Download Extract**:
  - Once the extract is ready, downloads the file from Oracle HCM Cloud.
  - Assign sch_documentID = documentID and g_logFileName = concat('HELM_EmergencyContact_',instanceId,'.'csv')
3. **Process Extracted Data**
  - **Scope**    
    - **Stage File**: Reads the downloaded file.
    - **Log Header**: Writes a log header for tracking.
    - **For Each Record**: Processes each record in the extracted file.
      - **Scope**
        - **Check if employee exists in Helm**
          - **if employee exists**:
              - Check if employee emergency contact exists by checking Contact Full Name. 🔴
                - **Creat Contact** if not exists via POST
                - **Update Contact** if exists via Patch
          - **Otherwise**
            - g_ErrorCount += 1, g_Message = 'Employee doesn't exists in Helm', g_Status = 'Error'   
      - **Error Handling**
        - g_ErrorCount += 1, g_Message = 'Local fault in Helm Scope', g_Status = 'Error' 🔴
      - **WriteLogFile**
        - Person Id = Contact Employee Person ID, Object Id = Contact Full name, Status = g_Status, Message = g_Message
    - **Logging**
      - Read Log File and write FTP log     
  - **Error Handling**: 🔴 Alex 
    - Logger : "Ignore - Due to empty file from HCM extract"
    - Assign : g_ErrorCount = 0, g_SuccessCount = 0, g_TotalCount = 0 🔴
4. **Notifications**
- **Send Notifications**:
  - **Success**: Sends a success notification if all records are processed/
  - **Error**: Sends an error notification if any failures occur.

5. **Final Steps**
- **Stop**: Ends the integration.
- **Global Error Handling**: Catches any unhandled errors and sends an error notification

## HELM REST API Structure

| Endpoint                          | Method | Purpose                               |
|-----------------------------------|--------|---------------------------------------|
| `/api/v1/jobs/users/FindUsers`    | GET    | Search for employees                  |      
| `/api/v2/public/emergencycontacts`| GET,POST,PATCH   | Search,Create and update emergency contact          |


**Key Notes:**
1. Error responses use `APIInvocationError` format


## Phase Variables

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

## SOAP Invocations to HCM
### 1. Submit Extract Request
**Purpose**: Initiate data extraction job for emergency contacts  
**Adapter**: `ORACLE_SOAP_HCM_EXTRACT`  
**Operation**: `submitAndGetFlowInstanceId`  
**Request Structure**:
```xml
<soap:Envelope>
  <soap:Body>
    <ns2:submitAndGetFlowInstanceId>
      <ns2:OutboundSOAPRequestDocument>
        <extractName>EMERGENCY_CONTACTS_EXTRACT</extractName>
        <parameters>
          <effectiveDate>{g_EffectiveDate}</effectiveDate>
          <processId>{g_Process_ID}</processId>
        </parameters>
      </ns2:OutboundSOAPRequestDocument>
    </ns2:submitAndGetFlowInstanceId>
  </soap:Body>
</soap:Envelope>
```
🔴 I couldn't the find the Extract Definitions in DEV3 and DEV4.
**Response**:
- Success: Returns flowInstanceId for tracking
- Failure: Throws ServiceException fault

### 2. Check Extract Status
**Purpose**: Poll extraction job status
**Adapter**: ORACLE_SOAP_HCM_EXTRACT
**Operation**: getIntegrationContentId
**Request Structure**:
```xml
<soap:Envelope>
  <soap:Body>
    <ns2:getIntegrationContentId>
      <ns2:OutboundSOAPRequestDocument>
        <flowInstanceId>{g_FlowInstanceStatus}</flowInstanceId>
      </ns2:OutboundSOAPRequestDocument>
    </ns2:getIntegrationContentId>
  </soap:Body>
</soap:Envelope>
```
**Status Values**:
- `COMPLETED`: Ready for download
- `IN_PROGRESS`: Continue polling
- `FAILED`: Terminate with error

### 3. Download Extract File
**Purpose**: Retrieve generated data file
**Adapter**: ORACLE_HCMCLOUD
**Operation**: GetFile
**Request Structure**:
```xml
<soap:Envelope>
  <soap:Body>
    <ns2:GetFile>
      <ns2:documentId>{sch_documentID}</ns2:documentId>
      <ns2:fileName>{g_FileName}</ns2:fileName>
    </ns2:GetFile>
  </soap:Body>
</soap:Envelope>
```
**Response**:
- CSV/XML file containing employee emergency contacts

## Mapping rules
| **Target Field**      | **Create Mapping Source**                                                                                       | **Update Mapping Source**                                                                                       |
|-----------------------|------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| PersonId              | `GetHELMEmployee.response-wrapper.Data.Page[1].Id`                                                              | *(Not used)*                                                                                                     |
| Id                    | *(Not used)*                                                                                                    | `GetContact.response-wrapper.Data.Results.Id`                                                                   |
| Name                  | `Extract_Contact_Full_Name`                                                                                      | `Extract_Contact_Full_Name`                                                                                      |
| PhoneNumber           | `Extract_Contact_Phone_Number`                                                                                   | `Extract_Contact_Phone_Number`                                                                                   |
| Relationship          | Based on `Extract_Contact_Type`: <br> `"S"` → Spouse, `"SISTER"` → Sister, `"P"` → Parent, etc.                 | *(Not included)*                                                                                                 |
| Address               | `Extract_Contact_Address_Line1`                                                                                  | `Extract_Contact_Address_Line1`                                                                                  |
| City                  | `Extract_Contact_Town_or_City`                                                                                   | `Extract_Contact_Town_or_City`                                                                                   |
| Country               | `Canada` *(hardcoded)*    ❗                                                                                       | `Canada` *(hardcoded)*   ❗                                                                                        |
| StateOrProvince       | `Extract_Contact_Address_Region3`                                                                                | `Extract_Contact_Address_Region3`                                                                                |
| PostalZipCode         | `Extract_Contact_Postal_Code`                                                                                    | `Extract_Contact_Postal_Code`                                                                                    |

# IMOHCM27_EMPLOYEE_MASTER_HELM
## Overview
The integration  utomates the synchronization of employee master data from Oracle HCM Cloud into the HELM system.
- Triggers on a scheduled basis.
- Reads employee changes from Oracle HCM Cloud via Atom Feeds and BI Publisher reports, capturing:
  - new hires
  - updates
  - assignment changes
  - terminations  ❗ This is missing!! ❗
- Transforms the data into HELM’s required format using XSLT mappings.
- Loads the data into HELM through REST API calls.
- Writes output files to SFTP servers for archiving or downstream processing.
- Sends notifications to alert stakeholders of successes or errors.

##  Integration Flow
1. **Initialization**
  - **Schedule Trigger**: The integration starts with a scheduled trigger with startTime as the tracking variable.
  - **Metadata Setup**: Initializes phase variables like startTime, g_RiceId, g_FlowName, g_FromEmail, g_ToEmail, etc.
  - **Stage File**: stage HeaderFile and EmpHeaderRec

2. **Read Atom Feeds from HCM**
   Four `parallel` Atom Feed adapters are configured:
   - **AtomFeedEmpUpdate**
   - **AtomFeedAssgnUpdate**
   - **AtomFeedNewHire**
   - **AtomFeedEmpTerminate**
   These adapters:
   - Query Atom Feeds to fetch incremental data since last run date (g_lastRunDate)
3. **For-Each Loop - Employee New Hire Records**

   Repeating Element : 
   ```xml
    /EmployeeNewHireFeedResponse/EmployeeNewHireFeed_update

   ```
   **Detailed Steps**:
   For each record:
     1. **stage file**: WriteNewHire
     2. **Invoke BI Report** : callEmployeeHELMRpt by Person Id 
         



   - - **Submit Extract Request**:
  - Transforms the request payload
  - Calls the HCM SOAP service to submit an extract request
- **Poll for Extract Status**:
  - Uses a while loop to periodically check the status of the extract. (g_FlowInstanceStatus = 'READY' or 'WAIT' or 'PAUSED' or 'RUNNING' ❗`Do we need READY?` )
  - Waits and checks the status via Check Extract Status SOAP call.
- **Download Extract**:
  - Once the extract is ready, downloads the file from Oracle HCM Cloud.
  - Assign sch_documentID = documentID and g_logFileName = concat('HELM_EmergencyContact_',instanceId,'.'csv')
