# HDL Error Handling Strategy for Payroll Integration

## 1. Recommended Strategy

The payroll integration involves sending full payroll batches in JSON format(Helm) or file format (Replicon) to Oracle Integration Cloud (OIC), which transforms and loads the data into Oracle HCM via HCM Data Loader (HDL). Due to the following business constraints:

* Boundary systems **must resend the entire batch** even if only a few records fail.
* There is **no callback mechanism** from the boundary systems
* **No Oracle costing API is available.**

The recommended strategy is:

<<<<<<< HEAD
=======
* Reject payload if not able to identify unique active assignment numbers otherwise **always send the full batch** to HDL, even if some records contain errors such as incorrect elementname
  * OIC sends error notification via email (given no callback API)
>>>>>>> 557733e2286604898cc0edd581a422b5d1a936f6
* **Use HDL as the authoritative validation engine** , relying on its native eligibility, costing, and business rules
* **Use a common utility (OIC_HCM_LOAD_IMPORT_DATA integration)** : This includes
  * Pick up the generated HDL zip file.
  * **Encrypt** and Invoke the HDL import and load process.
  * Monitor the process and generate a BI report based on the process ID.
  * Sending notifications for both success and error scenarios
<<<<<<< HEAD
* Use an ATP table for error tracebility (selected integratons only, for IT team use, no reporting; target for SIT2 )
=======
* Use ATP table for error tracking (target for SIT2). This includes
  * integration ID
  * instance ID
  * error message
* Use Object Storage for large payload and BI reports (TBD)
>>>>>>> 557733e2286604898cc0edd581a422b5d1a936f6

---

## 2. End-to-End Flow

**Payroll Batch Submission**

<<<<<<< HEAD
* Full payroll batch is submitted in:
  * JSON format(Helm): Sent to OIC, which stores it in SFTP.
  * File format(Replicon): submitted vis SFTP in one file (payroll+work order or payroll+project)
=======
* Full batch is submitted in JSON (Helm) or file format (Replcion)
  * File format (Replicon) :
    * Submitted via SFTP in one file (payroll/work order or payroll/project)
  * JSON payload (Helm) :
    * Three submission options
      1. Send payload directly to Object Storage which triggers OIC integration (POC required; recommended)
      2. Send payload to OIC, which stores it in:
         * Object Storage or
         * SFTP
>>>>>>> 557733e2286604898cc0edd581a422b5d1a936f6

**OIC Integration: Split (Replicon only),lookup and convert**

* **Lookup & Transformation** :

  * Identify a unique assignement number. **If unavailable, use the employee number.**
  * Transform the data into HDL-compatible format.
  * Generate an HDL zip file.

**OIC_HCM_LOAD_IMPORT_DATA Integration**

* Pick up the generated HDL zip file.
* **Encrypt** and Invoke the HDL import and load process in Oracle HCM.
* Monitor the process and generate a BI report based on the process ID. (**Connect to the offshore team?Need to know where to send**)
* **Notifications** :

  * Send success/error notifications via email.
  * For errors, attach the error report.
* **Error Handling** :

<<<<<<< HEAD
  * Log failed records in the ATP table (target for SIT2)
=======
  * Store the error report in Object Storage. (target for SIT2)
  * Log high level error details in the ATP table. (target for SIT2)
>>>>>>> 557733e2286604898cc0edd581a422b5d1a936f6

## 3. Benefits of HDL-Driven Validation

* **Native rules** : Validates eligibility, costing, and business rules configured in HCM
* **Maintainability** : Avoids duplicating logic in OIC; rules are centrally maintained in HCM
* **Partial processing** : Valid records are loaded even if some records fail
* **Consistent behavior** : Same validation used in manual HCM element entry and integrations
* **Auditability** : Generates structured reports for downstream analysis and reconciliation

---

## 4. Common HDL Errors

| **Category**  | **Sample Error Message**                             | **Cause**                               | **Resolution** |
| ------------------- | ---------------------------------------------------------- | --------------------------------------------- | -------------------- |
| Reference Error     | You must enter a valid assignment number.                  | AssignmentNumber not found or inactive.       | Business process     |
| Reference Error     | The element name is invalid or does not exist.             | ElementName is invalid                        | Business process     |
| Reference Error     | The input value name is not valid for the element.         | Invalid or misspelled input name.             | Business Process     |
| Business Rule Error | The element is not eligible for the employee's assignment. | Element is not available to the assignment.   | Business Process     |
| Business Rule Error | The input value is required.                               | Mandatory input (e.g., Pay Value) is missing. | Business Process     |
| Business Rule Risk  | *No error – duplicate entries are silently accepted*    | Same employee, element, and date repeated.    | Business Process     |

---

## 5. Business P10rocess

<<<<<<< HEAD
**Review error reports and process records**
=======
* Review error reports and process records
* Take corrective action based on the issue:
  - if resolvable in Oracle (e.g., missing configuration)
    - For issues affecting a small number of employees (<50), the business team can manually update records in Oracle
    - For larger-scale issues, request the Seaspan tech team to roll back and reprocess the entire payroll
  - If the issue originates from Helm (e.g., duplicate entries or incorrect calculations):
    - Notify the Seaspan tech team to roll back the payroll
    - Escalate to Helm to reprocess and resend the entire payroll
>>>>>>> 557733e2286604898cc0edd581a422b5d1a936f6

Take corrective action based on the issue:

<<<<<<< HEAD
1. **If issues need to be addressed in the system of records (Helm, Replicon)**

   * Oracle payroll team must roll back the payroll batch in Oracle.
   * Escalate to system of records to address the issues and resend the entire batch
2. **If resolution only required in Oracle** (e.g., missing configuration, inactive employee)

   * The business team must manually import the affected records as a resolution
=======
| **Concern Area**  | **Con of Pre-validation in OIC**                                             | **Impact**                                                            |
| ----------------------- | ---------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| Business Logic Accuracy | OIC cannot fully replicate validation  logic                                       | Risk of false results; HDL will still reject or allow based on actual rules |
| Maintainability         | Validation rules must be custom-coded and updated separately from HCM              | High cost and effort to keep logic in sync with HCM                         |
| Performance             | Pre-validating 30,000+ records with lookups is resource-intensive                  | May cause timeouts, slow runs, or unnecessary retries                       |
| Incomplete Detection    | OIC can't detect HDL-specific issues like DFF context violations                   | False sense of data quality; failures still happen in HDL                   |
| Complexity              | Adds control logic, conditional branches, and error checks to OIC                  | Harder to maintain, test, and trace integration flows                       |
| No Added Benefit        | OIC cannot reject bad records anyway – must send full batch to HDL | Extra work with no downstream benefit                                       |
| Duplicate Entry Risk    | OIC cannot access HCM history to detect prior entries                              | Duplicates silently accepted by HDL, causing risk of double pay             |
| No Callback API         | No Callback API for sending validation results back to the boundary systems        | Pre-validation errors must be handled via logging or manual communication   |
| Full Batch Resend       | HELM must resend full file even for 1–2 bad records                               | Pre-validation adds additional complexity                                   |
>>>>>>> 557733e2286604898cc0edd581a422b5d1a936f6
