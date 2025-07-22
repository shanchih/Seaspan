# HDL Error Handling Strategy for Payroll Integration

## 1. Recommended Strategy

The payroll integration involves sending full payroll batches in JSON format(Helm) or file format (Replicon) to Oracle Integration Cloud (OIC), which transforms and loads the data into Oracle HCM via HCM Data Loader (HDL). Due to the following business constraints:

* Boundary systems **must resend the entire batch** even if only a few records are bad
* There is **no callback mechanism** from the boundary systems
* **No Oracle costing API** 

The recommended strategy is:

* **Always send the full batch** to HDL, even if some records contain errors
* **Use HDL as the authoritative validation engine** , relying on its native eligibility, costing, and business rules
* **Parse and report errors** common utility to picks the generated HDL zip file and calls import and load HDL process. It also generates BI report based on process Id and send notification for success and error scenarios. 

---

## 2. End-to-End Flow



---

## 3. OIC Validation vs FBDI/HDL-Validation Comparison

| **Aspect**             | **OIC Validation**                        | **HDL validation**                                    |
| ---------------------------- | ------------------------------------------------------ | ------------------------------------------------------------------- |
| **Location**           | Logic built in OIC before submission                   | Let FBDI perform validations internally                              |
| **Performance**        | Slower (lookup-heavy, 30K+ records)                    | Faster bulk submission                                              |
| **Complexity**         | Requires complex logic and maintenance in OIC          | Simpler flows; relies on HDL/FBDI rules                                  |
| **Accuracy**           | Limited; can't enforce full HCM rules                  | Full validation including costing, eligibility, flexfields          |
| **Partial Success**    | Should not filter or reject in OIC | HDL supports partial success                                        |


---

## 4. Benefits of HDL-Driven Validation

* **Native rules** : Validates eligibility, costing, and business logic configured in HCM
* **Maintainability** : Avoids duplicating logic in OIC; rules maintained in one place (HCM)
* **Partial processing** : Valid records are loaded even if some records fail
* **Consistent behavior** : Same validation used in manual HCM element entry and integrations
* **Auditability** : Generates structured reports for downstream analysis and reconciliation

---

## 5. Common HDL Errors 

| **Category**  | **Sample Error Message**                                 | **Cause**                                    | **Resolution**                                         |
| ------------------- | -------------------------------------------------------------- | -------------------------------------------------- | ------------------------------------------------------------ |
| Reference Error     | You must enter a valid assignment number.                      | AssignmentNumber not found or inactive.            | Use a valid, active assignment number.                       |
| Reference Error     | The element name is invalid or does not exist.                 | ElementName is invalid or not assigned to the LDG. | Use an active, eligible element for the LDG.                 |
| Reference Error     | The input value name is not valid for the element.             | Invalid or misspelled input name.                  | Match input names to the element’s configuration.           |
| Reference Error     | The legal employer is invalid.                                 | LegalEmployerName is missing or incorrect.         | Confirm the legal employer in HCM.                           |
| Date Error          | You must enter a valid entry effective start date.             | Date missing or outside assignment bounds.         | Align dates with the employee's assignment.                  |
| Date Error          | The costing date is outside the element entry effective dates. | Costing date misaligned with entry dates.          | Match costing and entry dates.                               |
| Costing Error       | The account combination is invalid.                            | Invalid chart of accounts segment values.          | Validate the full account combination in GL setup.           |
| Costing Error       | You must specify a costing account for all levels.             | Required costing levels are missing.               | Include all necessary levels (e.g., Payroll, Org).           |
| Costing Error       | No costing rule defined for the assignment.                    | Costing is not set for the assignment or payroll.  | Ensure costing rules are set in HCM setup.                   |
| Business Rule Error | The element is not eligible for the employee's assignment.     | Element is not available to the assignment.        | Adjust element eligibility.                                  |
| Business Rule Error | The input value is required.                                   | Mandatory input (e.g., Pay Value) is missing.      | Provide all required input values.                           |
| Business Rule Error | You can't create overlapping entries for this element.         | Entry already exists with overlapping date.        | Avoid overlapping dates for the same element and assignment. |
| Business Rule Risk  | *No error – duplicate entries are silently accepted*        | Same employee, element, and date repeated.         | Payroll team must catch duplicates manually after load.      |
| Miscellaneous Error | The component ElementEntryId is required.                      | Updating without ElementEntryId.                   | Include ElementEntryId for updates.                          |
| Miscellaneous Error | The context combination is invalid.                            | DFF values missing or invalid.                     | Ensure required context values are passed.                   |

---

## 6. Cons of Pre-validation in OIC

| **Concern Area**  | **Con of Pre-validation in OIC**                                             | **Impact**                                                                       |
| ----------------------- | ---------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| Business Logic Accuracy | OIC cannot fully replicate validation  logic    | Risk of false results; HDL will still reject or allow based on actual rules            |
| Maintainability         | Validation rules must be custom-coded and updated separately from HCM              | High cost and effort to keep logic in sync with HCM                                    |
| Performance             | Pre-validating 30,000+ records with lookups is resource-intensive                  | May cause timeouts, slow runs, or unnecessary retries                                  |
| Incomplete Detection    | OIC can't detect HDL-specific issues like DFF context violations                   | False sense of data quality; failures still happen in HDL                              |
| Complexity              | Adds control logic, conditional branches, and error checks to OIC                  | Harder to maintain, test, and trace integration flows                                  |
| No Added Benefit        | OIC cannot reject bad records anyway – must send full batch per HELM requirements | Extra work with no downstream benefit                                                  |
| Duplicate Entry Risk    | OIC cannot access HCM history to detect prior entries                              | Duplicates silently accepted by HDL, causing risk of double pay                        |
| No Callback from HELM   | No API for sending validation results back to HELM in real-time                    | Pre-validation errors must be handled via logging or offline communication             |
| Full Batch Resend       | HELM must resend full file even for 1–2 bad records                               | Pre-validation adds complexity but doesn’t prevent HELM from resending unfixable data |
