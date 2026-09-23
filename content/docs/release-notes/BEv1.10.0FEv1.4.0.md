## **New Features & Enhancements**

### **Dashboard & Data Visualization**

* **Redesigned Dashboard Experience:** Overhauled the main dashboard view for all user types, featuring general and filtered overviews.  
* **Activity Timelines & Charts:** Added an activity timeline, composition bars, and lab overview graphs to the dashboard.  
* **Data Overview Section:** Introduced a comprehensive data overview section. ([Ticket \#262](https://github.com/azpathogens/apgap-development-tickets/issues/262), [Ticket \#269](https://github.com/azpathogens/apgap-development-tickets/issues/269), [Ticket \#276](https://github.com/azpathogens/apgap-development-tickets/issues/276))  
* **Data Explorer Presets:** Added support for storing, suggesting, updating, seeding, and switching data source presets. ([Ticket \#246](https://github.com/azpathogens/apgap-development-tickets/issues/246))


### **Navigation & UI**

* **Side Menu & Layout:** Rearranged the side menu for improved navigation. ([Ticket \#277](https://github.com/azpathogens/apgap-development-tickets/issues/277))  
* **Admin Context:** Platform overview context is now available in the switcher and navigation for administrators.  
* **Project List Card:** Redesigned the project list card and project list table view. ([Ticket \#274](https://github.com/azpathogens/apgap-development-tickets/issues/274))  
* **Lab Indicators:** Lab statuses are now visible on the lab picker, switcher, and across every lab page. ([Ticket \#283](https://github.com/azpathogens/apgap-development-tickets/issues/283))


### **Notifications & Alerts**

* **Notification Deep Linking:** Linked notifications directly to their target pages in the UI, updated email link wording, and made notification filtering available by unread state and status. ([Ticket \#265](https://github.com/azpathogens/apgap-development-tickets/issues/265))  
* **Lab Notification Filtering:** Enabled notification filtering by specific labs. ([Ticket \#267](https://github.com/azpathogens/apgap-development-tickets/issues/267))  
* **Project Alerts:** Automated notifications for project creation failures and successful builds. ([Ticket \#284](https://github.com/azpathogens/apgap-development-tickets/issues/284))  
* **Markdown System Alerts:** Switched the system alert editor to Markdown and added explicit styling for rendered Markdown banners. ([Ticket \#281](https://github.com/azpathogens/apgap-development-tickets/issues/281))

### **Backend & API**

* **API Exposing:** Exposed Seqera workspace and notebook activity parameters directly on project serializers.  
* **Lab Recent Activity Endpoint:** Added a dedicated API endpoint for retrieving recent lab activity.  
* **CLI Safety Guards:** Added confirmation prompts and safety guards when using the \--clear command.


## **Bug Fixes**

* **Project Notebook Controls:** Allow project users to start and stop notebooks directly.  
* **Lab Context & Dropdowns:**

  * Excluded inactive labs from the dataset creation lab dropdown. ([Ticket \#282](https://github.com/azpathogens/apgap-development-tickets/issues/282))  
  * Resolved issues where the lab picker falls back to the loaded lab when no lab is actively selected.  
  * Reconciled admin lab context options with flat lab navigation.

* **User Management:** Automatically filter out inactive users across the system.  
* **Batch Endpoint Action:** Updated the revoke action UI on batch endpoints to use a trash icon. ([Ticket \#247](https://github.com/azpathogens/apgap-development-tickets/issues/247))  
* **Configuration Fixes:** Ensured production environments default to DEBUG \= False and set is\_pathogen\_key to bootstrap automatically.
