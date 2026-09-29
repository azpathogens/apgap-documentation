+++
title = 'BE v1.11.0, FE v1.5.0'
date = 2026-09-25T07:07:07+01:00
weight = 11
+++

## **Features**

* **Acceptable Use Policy (AUP)**  
  * Gate the application behind mandatory AUP acknowledgment ([Ticket \#230](https://github.com/azpathogens/apgap-development-tickets/issues/230))  
  * Add ability to print a list of users and their accepted AUPs ([Ticket \#294](https://github.com/azpathogens/apgap-development-tickets/issues/294))  
* **Project Management**  
  * Add a toggle on the Projects page to show archived projects ([Ticket \#298](https://github.com/azpathogens/apgap-development-tickets/issues/298))  
  * List archived projects without granting access ([Ticket \#298](https://github.com/azpathogens/apgap-development-tickets/issues/298))  
* **Lab Archiving**  
  * Add button to request archiving a lab ([Ticket \#287](https://github.com/azpathogens/apgap-development-tickets/issues/287), [Ticket \#86](https://github.com/azpathogens/apgap-development-tickets/issues/86))  
* **Dashboards & Action Items**  
  * Introduce Action Items ([Ticket \#279](https://github.com/azpathogens/apgap-development-tickets/issues/279))  
  * Replace dashboard error notifications with actionable items ([Ticket \#279](https://github.com/azpathogens/apgap-development-tickets/issues/279))  
* **Notifications**  
  * Include notifications with no assigned lab as part of the lab filters  
  * Consistently clear highlighted tabs when navigating away from the dashboard ([Ticket \#268](https://github.com/azpathogens/apgap-development-tickets/issues/268))  
  * Reworked mark-as-read notification behavior  
  * Prevent page flash when clicking the notification bell  
* **Uploads & Endpoints**  
  * Add maximum recommended file size limit and display warning on upload/batch endpoints  
  * Improve batch endpoints page ([Ticket \#275](https://github.com/azpathogens/apgap-development-tickets/issues/275))

## **Bug Fixes**

* **Notifications & Access Requests**  
  * Tag new access request notices with the associated file lab ([Ticket \#302](https://github.com/azpathogens/apgap-development-tickets/issues/302))  
  * Prevent repeated access request notifications ([Ticket \#293](https://github.com/azpathogens/apgap-development-tickets/issues/293))  
  * Tag access request reminders with the approver's lab  
  * Update notification text content  
* **Interface & Styling**  
  * Fix broken toggle styles ([Ticket \#295](https://github.com/azpathogens/apgap-development-tickets/issues/295))  
  * Enhance visibility of Data Explorer presets ([Ticket \#246](https://github.com/azpathogens/apgap-development-tickets/issues/246))  
  * Correct project users count ([Ticket \#297](https://github.com/azpathogens/apgap-development-tickets/issues/297))  
  * Revalidate cached HTML when loading the application  
* **Documentation & Data**  
  * Update User Guide link to point to [docs.azpathogens.org](https://docs.azpathogens.org) ([Ticket \#301](https://github.com/azpathogens/apgap-development-tickets/issues/301))  
  * Update populate dummy data script  
  * Skip lab tag when an option request spans multiple labs



