# IMOHCM27_HELM_HCM_EMPLOYEE_MASTER_OUT
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


## SOAP Invocations to HCM
### 1. Submit Extract Request
**Purpose**: Initiate data extraction job for emergency contacts  
**Adapter**: `ORACLE_SOAP_HCM_EXTRACT`  
**Operation**: `submitAndGetFlowInstanceId`  

🔴 I couldn't the find the Extract Definitions in DEV3 and DEV4.
**Response**:
- Success: Returns flowInstanceId for tracking
- Failure: Throws ServiceException fault

### 2. Check Extract Status
**Purpose**: Poll extraction job status
**Adapter**: ORACLE_SOAP_HCM_EXTRACT
**Operation**: getIntegrationContentId

**Status Values**:
- `COMPLETED`: Ready for download
- `IN_PROGRESS`: Continue polling
- `FAILED`: Terminate with error

### 3. Download Extract File
**Purpose**: Retrieve generated data file
**Adapter**: ORACLE_HCMCLOUD
**Operation**: GetFile
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
  - terminations 
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
     3. **Stage File**: write and read report
     4. For reach G_1
        - Invoke Helm jobs/users/FindUsers (GET) using EmployeeNumber
        - Map Id from Helm
        - Create or Update User using Helm jobs/users/CreateOrUpdateUser (POST)
        - **Stage File** : stage record from BI report (If count(G_1) >0 ) 
         
4. **For-Each Loop - Employee Update Records**  ❗ (Alex) Implementation is not correct ❗

   Repeating Element : 
   ```xml
    /EmployeeFeedResponse/EmployeeUpdateFeed_update

   ```
   **Detailed Steps**:

     - For each EmployeeUpdate:

       ```xml
       /EmployeeFeedResponse/EmployeeUpdateFeed_update
       ```
       - For each Changed Attributes:
         ```xml
         /EmployeeUpdateFeed_update/ChangedAttributes
         ```
         - **Switch**
         AttributeName = 'LastName', 'FirstName' or 'PersonNumber' or 'WorkEmail or 1 = 1  
           - **stage file**: WriteEmpFile
           - Invoke Helm jobs/users/FindUsers (GET) using EmployeeNumber
           - Map Id from Helm
           - Create or Update User using Helm jobs/users/CreateOrUpdateUser (POST)
          
5. **Employee Assignment Records**  ❗ (Alex) Implementation is not correct
  - **Invoke BI Report** : callEmployeeHELMRpt by Person Id
  - **Stage File**: write and read report
  - For each G_1
    - Get User /jobs/users/FindUsers
      - If user exists, Update Helm /job/users/CreateOrUpdateUser via POST  

6. **For-Each Loop - Employee Termination Records**  

   Repeating Element : 
   ```xml
    /EmployeeTerminationFeedResponse/EmployeeTerminationFeed_update

   ```
   **Detailed Steps**:
   For each record:
     1. **stage file**: stage termination file
     2. **Invoke Helm API** : Get /jobs/users/FindUsers
     3. **Invoke Helm API**: POST /job/users/CreateOrUpdateUser

##  Mapping rules


| Target Element          | Source                                                                 | Mapping Rule                                                                 |
|-------------------------|------------------------------------------------------------------------|------------------------------------------------------------------------------|
| id                      | `getRecords/executeResponse/response-wrapper/Data/Page/Id`             | Only mapped if `TotalCount > 0`                                              |
| FirstName               | `f1_EmpUpdateChangedAttributes/ChangedAttributes/NewValue`             | When `AttributeName="FirstName"` exists with a `NewValue`                    |
|                         | `getRecords/executeResponse/response-wrapper/Data/Page/FirstName`      | Fallback when no FirstName change is detected                                |
| PreferredName           | `f1_EmpUpdateChangedAttributes/ChangedAttributes/NewValue`             | When `AttributeName="KnownAs"` exists with a `NewValue`                      |
|                         | `getRecords/executeResponse/response-wrapper/Data/Page/PreferredName`  | Fallback when no KnownAs change is detected                                  |
| MiddleName              | `f1_EmpUpdateChangedAttributes/ChangedAttributes/NewValue`             | When `AttributeName="MiddleNames"` exists with a `NewValue`                  |
|                         | `getRecords/executeResponse/response-wrapper/Data/Page/MiddleName`     | Fallback when no MiddleNames change is detected                              |
| LastName                | `f1_EmpUpdateChangedAttributes/ChangedAttributes/NewValue`             | When `AttributeName="LastName"` exists with a `NewValue`                     |
|                         | `getRecords/executeResponse/response-wrapper/Data/Page/LastName`       | Fallback when no LastName change is detected                                 |
| Email                   | `f1_EmpUpdateChangedAttributes/ChangedAttributes/NewValue`             | When `AttributeName="WorkEmail"` exists with a `NewValue`                    |
|                         | `getRecords/executeResponse/response-wrapper/Data/Page/Email`          | Fallback when no WorkEmail change is detected                                |
| EmployeeNumber          | `f0_EmployeeUpdateFeed_Update/EmployeeUpdateFeed_Context/PersonNumber` | Direct mapping                                                               |
| CellPhoneNumber         | `f1_EmpUpdateChangedAttributes/ChangedAttributes/NewValue`             | When `AttributeName="PhoneNumber"` exists with a `NewValue`                  |
|                         | `getRecords/executeResponse/response-wrapper/Data/Page/CellPhoneNumber`| Fallback when no PhoneNumber change is detected                              |
| CanLogIn                | Static value                                                           | Always set to "true"                                                         |
| IsActiveEmployee        | Static value                                                           | Always set to "true"                                                         |
| Division/Name           | Static value                                                           | Always set to "Ferries"                                                      |



| Target Element                          | Source                                                                 | Mapping Rule                                                                 |
|-----------------------------------------|------------------------------------------------------------------------|------------------------------------------------------------------------------|
| HTTPHeaders/API-Key                     | Static value                                                           | Hardcoded API key (redacted for security)                                    |
| HTTPHeaders/Content-Type                | Static value                                                           | Always set to "application/json"                                             |
| Id                                      | `getNewHireUser/executeResponse/response-wrapper/Data/Page/Id`         | Only mapped if value is not empty                                            |
| FirstName                               | `forEach_G_1/G_1/FIRST_NAME`                                          | Direct mapping                                                               |
| PreferredName                           | `forEach_G_1/G_1/PREFERRED_NAME`                                      | Direct mapping                                                               |
| MiddleName                              | `forEach_G_1/G_1/MIDDLE_NAMES`                                        | Direct mapping                                                               |
| LastName                                | `forEach_G_1/G_1/LAST_NAME`                                           | Direct mapping                                                               |
| Email                                   | `forEach_G_1/G_1/WORK_EMAIL`                                          | Direct mapping                                                               |
| EmployeeNumber                          | `forEach_G_1/G_1/PERSON_NUMBER`                                       | Direct mapping                                                               |
| CellPhoneNumber                         | `forEach_G_1/G_1/USERCELLPHONENUMBER`                                 | Direct mapping                                                               |
| CanLogIn                                | `forEach_G_1/G_1/CANLOGIN`                                            | Direct mapping                                                               |
| IsActiveEmployee                        | `forEach_G_1/G_1/ISACTIVEEMPLOYEE`                                    | Direct mapping                                                               |
| UserDefined/AssignmentNumber            | `forEach_G_1/G_1/ASSIGNMENT_NUMBER`                                   | Direct mapping                                                               |
| UserDefined/AssignmentStatus            | Static value                                                           | Always set to "Active - Payroll Eligible"                                    |
| UserDefined/CarriedDate                 | `forEach_G_1/G_1/CARRIEDDATEEIT`                                      | Only mapped if value is not empty                                            |
| UserDefined/CategorySeniorityDate       | `forEach_G_1/G_1/ORIGINALHIREDATE`                                    | Only mapped if value is not empty                                            |
| UserDefined/PersonCollectiveAgreement   | `forEach_G_1/G_1/COLLECTIVEAGREEMENT`                                 | Direct mapping                                                               |
| UserDefined/EnteredServiceDate          | `forEach_G_1/G_1/ENTEREDSERVICEDATEEFF`                               | Only mapped if value is not empty                                            |
| UserDefined/GradeStep                   | `forEach_G_1/G_1/GRADE_STEP_NAME`                                     | Direct mapping                                                               |
| UserDefined/OvertimeBank                | `forEach_G_1/G_1/OT_BANK_ASG_DFF`                                     | Direct mapping                                                               |
| UserDefined/ServiceSeniorityDate        | `forEach_G_1/G_1/SERVICESENIORITYDATEEFF`                             | Only mapped if value is not empty                                            |
| UserDefined/Union                       | `forEach_G_1/G_1/UNIONEMP`                                            | Direct mapping                                                               |
| UserDefined/VacationPaidOrAccrued       | `forEach_G_1/G_1/VACPAIDORACCR_ASG_DFF`                               | Direct mapping                                                               |
| PayrollClassHistory/Upsert/EffectiveDate| Current timestamp                                                     | Formatted as YYYY-MM-DDTHH:MM:SS                                             |
| PayrollClassHistory/Upsert/PayrollClass/Name | `forEach_G_1/G_1/PAYROLLCLASS`                                   | Direct mapping                                                               |
| PayrollGroup/Name                       | `forEach_G_1/G_1/PAYROLLGROUP`                                        | Direct mapping                                                               |
| Division/Name                           | `forEach_G_1/G_1/LEGAL_EMPLOYER`                                      | Direct mapping                                                               |
| Positions/Name                          | `forEach_G_1/G_1/POSITION`                                            | Direct mapping                                                               |
| Department/Name                         | `forEach_G_1/G_1/POSITION`                                            | Direct mapping (same as Positions/Name)                                      |


| Target Element                          | Source                                                                 | Mapping Rule                                                                 |
|-----------------------------------------|------------------------------------------------------------------------|------------------------------------------------------------------------------|
| HTTPHeaders/API-Key                     | Static value                                                           | Hardcoded API key (redacted for security)                                    |
| id                                      | `f9_Page/Page/Id`                                                     | Only mapped if value is not empty                                            |
| FirstName                               | `forEachEmp_G_1/G_1/FIRST_NAME`                                       | When not empty, otherwise falls back to `f9_Page/Page/FirstName`             |
| PreferredName                           | `forEachEmp_G_1/G_1/PREFERRED_NAME`                                   | When not empty, otherwise falls back to `f9_Page/Page/PreferredName`         |
| MiddleName                              | `forEachEmp_G_1/G_1/MIDDLE_NAMES`                                     | When not empty, otherwise falls back to `f9_Page/Page/MiddleName`            |
| LastName                                | `forEachEmp_G_1/G_1/LAST_NAME`                                        | When not empty, otherwise falls back to `f9_Page/Page/LastName`              |
| Email                                   | `forEachEmp_G_1/G_1/WORK_EMAIL`                                       | When not empty, otherwise falls back to `f9_Page/Page/Email`                 |
| EmployeeNumber                          | `forEachEmp_G_1/G_1/PERSON_NUMBER`                                    | Direct mapping                                                               |
| CanLogIn                                | Static value                                                           | Always set to "yes"                                                          |
| IsActiveEmployee                        | Static value                                                           | Always set to "yes"                                                          |
| UserDefined/AssignmentNumber            | `forEachEmp_G_1/G_1/ASSIGNMENT_NUMBER`                                | When not empty, otherwise falls back to `f9_Page` source                     |
| UserDefined/AssignmentStatus            | `forEachEmp_G_1/G_1/ASSIGNMENT_STATUS`                                | When not empty, otherwise falls back to `f9_Page` source                     |
| UserDefined/PersonCollectiveAgreement   | `forEachEmp_G_1/G_1/COLLECTIVEAGREEMENT`                              | When not empty, otherwise falls back to `f9_Page` source                     |
| UserDefined/GradeStep                   | `forEachEmp_G_1/G_1/GRADE_STEP_NAME`                                  | When not empty, otherwise falls back to `f9_Page` source                     |
| UserDefined/MidMonthAdvance             | `forEachEmp_G_1/G_1/MIDMONTHADVANCE_DFF`                              | When not empty, otherwise falls back to `f9_Page` source                     |
| UserDefined/OvertimeBank                | `forEachEmp_G_1/G_1/OT_BANK_ASG_DFF`                                  | When not empty, otherwise falls back to `f9_Page` source                     |
| UserDefined/StatDayBank                 | Static value                                                           | Always set to "Y"                                                            |
| UserDefined/Union                       | `forEachEmp_G_1/G_1/UNIONEMP`                                         | When not empty, otherwise falls back to `f9_Page` source                     |
| UserDefined/UseMidMonthAdvanceOverride  | Static value                                                           | Always set to "false"                                                        |
| UserDefined/VacationPaidOrAccrued       | `forEachEmp_G_1/G_1/VACPAIDORACCR_ASG_DFF`                            | When not empty, otherwise falls back to `f9_Page` source                     |
| Division/Name                           | Static value                                                           | Always set to "Ferries"                                                      |



| Target Element                  | Source                                                                 | Mapping Rule                                                                 |
|---------------------------------|------------------------------------------------------------------------|------------------------------------------------------------------------------|
| HTTPHeaders/API-Key             | Static value                                                           | Hardcoded API key (redacted)                                                 |
| HTTPHeaders/Content-Type        | Static value                                                           | Always "application/json"                                                    |
| id                              | `f10_Page/Page/Id`                                                    | Only if value exists                                                         |
| FirstName                       | `f11_ChangedAttributes` (when AttributeName="FirstName")               | Uses changed value if exists, otherwise falls back to `f10_Page/Page/FirstName` |
| PreferredName                   | `f11_ChangedAttributes` (when AttributeName="KnownAs")                 | Uses changed value if exists, otherwise falls back to `f10_Page/Page/PreferredName` |
| MiddleName                      | `f11_ChangedAttributes` (when AttributeName="MiddleNames")             | Uses changed value if exists, otherwise falls back to `f10_Page/Page/MiddleName` |
| LastName                        | `f11_ChangedAttributes` (when AttributeName="LastName")                | Uses changed value if exists, otherwise falls back to `f10_Page/Page/LastName` |
| Email                           | `f11_ChangedAttributes` (when AttributeName="WorkEmail")               | Uses changed value if exists, otherwise falls back to `f10_Page/Page/Email`  |
| EmployeeNumber                  | `f6_EmployeeTerminationFeed_Update/EmployeeTerminationFeed_Context/PersonNumber` | Direct mapping                                                      |
| CanLogIn                        | Static value                                                           | Always "false"                                                               |
| CanLogInToBoat                  | Static value                                                           | Always "false"                                                               |
| CanLogInToShore                 | Static value                                                           | Always "false"                                                               |
| IsActiveEmployee                | Based on termination date                                              | "false" if termination date exists in changes, otherwise "true"              |
| UserDefined/AssignmentNumber    | `f10_Page/Page/UserDefined/AssignmentNumber`                           | Direct mapping                                                               |
| UserDefined/AssignmentStatus    | `f10_Page/Page/UserDefined/AssignmentStatus`                           | Direct mapping                                                               |
| UserDefined/TerminationDate     | `f11_ChangedAttributes` (when AttributeName="ActualTerminationDate")   | Uses changed value if exists, otherwise from `f10_Page` source               |
| Division/Name                   | Static value                                                           | Always "Ferries"                                                             |


# IMOHCM27_NON_MARINE_USERS_HELM
## Overview
The integration IMOHCM27_NON_MARINE_USERS_HELM orchestrates a process to extract user data for non-marine employees from Oracle HCM Cloud and transmit it to the HELM system.  🔴 How to distinquish non-marine employees
  - Triggering an HCM extract via SOAP API 
  - Monitoring the extract process using getFlowInstanceStatus
  - Downloading the resulting file from Oracle HCM
  - Mapping and transforming the data using XSLT to the HELM payload format
  - Sending employee records to HELM through REST APIs 
  - Handling success and error notifications via HELM endpoints
  - Logging details to SFTP for audit or troubleshooting

## mapping rules

| Target Element          | Source                                                                 | Mapping Rule                                                                 |
|-------------------------|-----------------------------------------------------------------------|------------------------------------------------------------------------------|
| id                      | `$f1_Page/ns36:Page/ns36:Id`                                         | Only mapped if `$GetHELMEmployee/nsmpr1:executeResponse/ns36:response-wrapper/ns36:Data/ns36:TotalCount != 0` |
| FirstName               | `$for_Each/nsmpr3:MAIN_DG/nsmpr3:EMPLOYEE_REC/nsmpr3:FirstName`      | Direct mapping from source to target.                                        |
| LastName                | `$for_Each/nsmpr3:MAIN_DG/nsmpr3:EMPLOYEE_REC/nsmpr3:LastName`       | Direct mapping from source to target.                                        |
| Email                   | `$for_Each/nsmpr3:MAIN_DG/nsmpr3:EMPLOYEE_REC/nsmpr3:Email`          | Direct mapping from source to target.                                        |
| EmployeeNumber          | `$for_Each/nsmpr3:MAIN_DG/nsmpr3:EMPLOYEE_REC/nsmpr3:EmployeeNumber` | Direct mapping from source to target.                                        |
| CanLogIn                | `$for_Each/nsmpr3:MAIN_DG/nsmpr3:EMPLOYEE_REC/nsmpr3:AssignmentStatus` | Set to `true` if AssignmentStatus is one of: "Leave of Absence – Payroll Eligible", "Active - Payroll Eligible", or "Inactive - Payroll Eligible"; otherwise `false`. |
| CanLogInToShore         | `$for_Each/nsmpr3:MAIN_DG/
