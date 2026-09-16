+++
title = 'BE v1.9.0, FE v1.3.0'
date = 2026-09-16T07:07:07+01:00
weight = 11
+++

## **Changelog (Backend v1.9.0 / Frontend v1.3.0)**

### New Features

* **Batch Operations:** Added an option to revoke batch and batch upload endpoints ([Ticket \#247](https://github.com/azpathogens/apgap-development-tickets/issues/247)).  
* **Seqera Integration:** Improved error message reporting and capture when Seqera operations fail ([Ticket \#249](https://github.com/azpathogens/apgap-development-tickets/issues/249)).  
* **Dataset Management:**  
  * Enabled dynamic dataset backfilling alongside corrected dataset previews ([Ticket \#260](https://github.com/azpathogens/apgap-development-tickets/issues/260)).Added manual retry support for dataset file copying ([Ticket \#227](https://github.com/azpathogens/apgap-development-tickets/issues/227)).  
* **Pipeline Updates:**  
  * Added ability to view and filter pipelines using topic tags ([Ticket \#255](https://github.com/azpathogens/apgap-development-tickets/issues/255)).  
  * Pipelines page now includes forked repositories.  
* **Project Management:** Surfaced "cancelled" and "timed out" project statuses and added the ability to delete affected projects.

### Bug Fixes & Security

* **Notifications:** Distinguished badge reminder notifications and explicitly included the timestamp of when requests were made ([Ticket \#266](https://github.com/azpathogens/apgap-development-tickets/issues/266)).  
* **Session & Account Security:** Ensured the NgRx store resets during session changes to prevent lab state data from leaking between accounts ([Ticket \#273](https://github.com/azpathogens/apgap-development-tickets/issues/273)).  
* **User Management:** Added automatic refresh for current user data following changes to their own lab or project roles ([Ticket \#264](https://github.com/azpathogens/apgap-development-tickets/issues/264)).  
* **Error Handling:** Workspace URL errors for Seqera are now properly displayed ([Ticket \#249](https://github.com/azpathogens/apgap-development-tickets/issues/249)).
