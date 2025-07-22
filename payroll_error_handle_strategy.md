# HDL Error Handling Strategy for Payroll Integration

## 1. Recommended Strategy

The payroll integration involves sending full payroll batches in JSON format(Helm) or file format (Replicon) to Oracle Integration Cloud (OIC), which transforms and loads the data into Oracle HCM via HCM Data Loader (HDL). Due to the following business constraints:

* Boundary systems **must resend the entire batch** even if only a few records are bad
* There is **no callback mechanism** from the boundary systems
* **No Oracle costing API**

The recommended strategy is:

* Reject payload if not able to identify unique active assignment numbers otherwise **always send the full batch** to HDL, even if some records contain errors such as incorrect elementname
  * How OIC notifiy the boundary system? through email?
* **Use HDL as the authoritative validation engine** , relying on its native eligibility, costing, and business rules
* Use a common utility (OIC_HCM_LOAD_IMPORT_DATA integration) : This includes 
    * Picking up the generated HDL zip file
    * Invoking the HDL import and load process
    * Monitoring the process and generating a BI report based on the process ID
    * Sending notifications for both success and error scenarios
* Use ATP table for error tracking (target for SIT2). This includes 
  * integration ID 
  * instance ID 
  * error message 
  

---

## 2. End-to-End Flow

---

## 3. OIC Validation vs HDL-Validation Comparison

| **Aspect**          | **OIC Validation**                      | **HDL validation**                                   |
| ------------------------- | --------------------------------------------- | ---------------------------------------------------------- |
| **Location**        | Logic built in OIC before submission          | validation handled internally by HDL                       |
| **Performance**     | Slower (lookup-heavy, 30K+ records)           | Faster bulk submission                                     |
| **Complexity**      | Requires complex logic and maintenance in OIC | Simpler flows; relies on HDL/FBDI rules                    |
| **Accuracy**        | Limited; can't enforce full HCM rules         | Full validation including costing, eligibility, flexfields |
| **Partial Success** | Should not filter or reject in OIC (reject payroll if not able to find assignment number) ?          | HDL supports partial success                               |

---

## 4. Benefits of HDL-Driven Validation

* **Native rules** : Validates eligibility, costing, and business rules configured in HCM
* **Maintainability** : Avoids duplicating logic in OIC; rules are centrally maintained in HCM
* **Partial processing** : Valid records are loaded even if some records fail
* **Consistent behavior** : Same validation used in manual HCM element entry and integrations
* **Auditability** : Generates structured reports for downstream analysis and reconciliation

---

## 5. Common HDL Errors

| **Category**  | **Sample Error Message**                                 | **Cause**                                    | **Resolution**                                         |
| ------------------- | -------------------------------------------------------------- | -------------------------------------------------- | ------------------------------------------------------------ |
| Reference Error     | You must enter a valid assignment number.                      | AssignmentNumber not found or inactive.            | Rejects payload.                       |
| Reference Error     | The element name is invalid or does not exist.                 | ElementName is invalid | Business process                 |
| Reference Error     | The input value name is not valid for the element.             | Invalid or misspelled input name.                  | Business Process           |
| Business Rule Error | The element is not eligible for the employee's assignment.     | Element is not available to the assignment.        | Business Process                                  |
| Business Rule Error | The input value is required.                                   | Mandatory input (e.g., Pay Value) is missing.      | Business Process                         |
| Business Rule Risk  | *No error – duplicate entries are silently accepted*        | Same employee, element, and date repeated.         | Business Process      |

---

## 7. Business Process
* Review error reports and process records
* Take corrective action based on the issue:
  - if resolvable in Oracle (e.g., missing configuration)
    - For issues affecting a small number of employees (<50>), the business team can manually update records in Oracle
    - For larger-scale issues, request the Seaspan tech team to roll back and reprocess the entire payroll
  - If the issue originates from Helm (e.g., duplicate entries or incorrect calculations):
    - Notify the Seaspan tech team to roll back the payroll
    - Escalate to Helm to reprocess and resend the entire payroll


## 8. Cons of Pre-validation in OIC

| **Concern Area**  | **Con of Pre-validation in OIC**                                             | **Impact**                                                            |
| ----------------------- | ---------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| Business Logic Accuracy | OIC cannot fully replicate validation  logic                                       | Risk of false results; HDL will still reject or allow based on actual rules |
| Maintainability         | Validation rules must be custom-coded and updated separately from HCM              | High cost and effort to keep logic in sync with HCM                         |
| Performance             | Pre-validating 30,000+ records with lookups is resource-intensive                  | May cause timeouts, slow runs, or unnecessary retries                       |
| Incomplete Detection    | OIC can't detect HDL-specific issues like DFF context violations                   | False sense of data quality; failures still happen in HDL                   |
| Complexity              | Adds control logic, conditional branches, and error checks to OIC                  | Harder to maintain, test, and trace integration flows                       |
| No Added Benefit        | OIC cannot reject bad records anyway – must send full batch per HELM requirements | Extra work with no downstream benefit                                       |
| Duplicate Entry Risk    | OIC cannot access HCM history to detect prior entries                              | Duplicates silently accepted by HDL, causing risk of double pay             |
| No Callback API         | No Callback API for sending validation results back to the boundary systems        | Pre-validation errors must be handled via logging or manual communication   |
| Full Batch Resend       | HELM must resend full file even for 1–2 bad records                               | Pre-validation adds complexity                                              |
