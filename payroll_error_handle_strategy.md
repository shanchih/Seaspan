# HDL Error Handling Strategy for Payroll Integration

## 1. Recommended Strategy

The payroll integration involves sending full payroll batches in JSON format(Helm) or file format (Replicon) to Oracle Integration Cloud (OIC), which transforms and loads the data into Oracle HCM via HCM Data Loader (HDL). Due to the following business constraints:

* Boundary systems **must resend the entire batch** even if only a few records fail.
* There is **no callback mechanism** from the boundary systems
* **No Oracle costing API is available.**

The recommended strategy is:

* **Use HDL as the authoritative validation engine** , relying on its native eligibility, costing, and business rules
* **Use a common utility (OIC_HCM_LOAD_IMPORT_DATA integration)** : This includes
  * Pick up the generated HDL zip file.
  * **Encrypt** and Invoke the HDL import and load process.
  * Monitor the process and generate a BI report based on the process ID.
  * Sending notifications for both success and error scenarios
* Use an ATP table for error tracebility (selected integratons only, for IT team use, no reporting; target for SIT2 )

---

## 2. End-to-End Flow

**Payroll Batch Submission**

* Full payroll batch is submitted in:
  * JSON format(Helm): Sent to OIC, which stores it in SFTP.
  * File format(Replicon): submitted vis SFTP in one file (payroll+work order or payroll+project)

**OIC Integration: Split (Replicon only),lookup and convert**

* **Lookup & Transformation** :

  * Identify a unique assignement number. **If unavailable, use the employee number.**
  * Transform the data into HDL-compatible format.
  * Generate an HDL zip file.

**`OIC_HCM_LOAD_IMPORT_DATA` Integration**

* Pick up the generated HDL zip file.
* **Encrypt** and Invoke the HDL import and load process in Oracle HCM.
* Monitor the process and generate a BI report based on the process ID. (**Connect to the offshore team?Need to know where to send**)
* **Notifications** :

  * Send success/error notifications via email.
  * For errors, attach the error report.
* **Error Handling** :

  * Log failed records in the ATP table (target for SIT2)

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

**Review error reports and process records**

Take corrective action based on the issue:

1. **If issues need to be addressed in the system of records (Helm, Replicon)**

   * Oracle payroll team must roll back the payroll batch in Oracle.
   * Escalate to system of records to address the issues and resend the entire batch
2. **If resolution only required in Oracle** (e.g., missing configuration, inactive employee)

   * The business team must manually import the affected records as a resolution
