
---

# **GeneSys Research Suite – System Overview**

*Internal Documentation – Draft v1.0*
*Author: Hashim Zaffar*

## **1. Introduction**

The GeneSys Research Suite is an on-premise platform used by life science laboratories to manage sample tracking, sequencing workflows, and data analysis. Each client lab receives its own dedicated server and isolated deployment. This document provides a high-level overview of the system architecture, components, data flows, and operational behavior. It is intended for Support, Customer Success (CS), and QA teams who interact with the environment daily.

This overview consolidates historical notes, internal messages, and configuration logs into a unified reference. Assumptions made for clarity are noted where relevant.

---

## **2. System Architecture Overview**

GeneSys follows a **single-tenant**, on-prem model. Each lab receives its own:

* Windows Server (self-contained environment)
* SQL Server instance
* IIS-hosted web applications
* BioBridge/BioSync Service

This isolation ensures data privacy, reduces cross-tenant risk, and provides flexibility for labs with different product licenses.

**[DIAGRAM PLACEHOLDER – High-Level Architecture Diagram]**

---

## **3. Core Components**

### **3.1 GeneSys Core (Mandatory Component)**

GeneSys Core is the central desktop application used by scientists to:

* Record samples and sequencing runs
* Manage experiment metadata
* Maintain the authoritative SQL database

**Key characteristics:**

| Attribute      | Details                  |
| -------------- | ------------------------ |
| Hosting        | Local Windows Server     |
| Database       | Microsoft SQL Server     |
| Authentication | Windows Authentication   |
| Availability   | Required for all clients |

Core acts as the main source of truth. All downstream systems reference data originating from this database.

---

### **3.2 Web Applications (Optional Components)**

All web applications run in **IIS** and each uses different authentication and data flow patterns.

#### **GeneSys Portal**

* Used by researchers to submit sequencing requests
* Integrates directly with Core via REST API
* Does **not** rely on BioBridge
* Authentication: OAuth

#### **GeneSys Insight**

* Provides analytics dashboards and experiment visualizations
* Depends on BioBridge for hourly data syncs from Core
* Authentication: API Keys

#### **GeneSys Share**

* Used to share approved laboratory data externally
* Cannot access live Core database; relies on a nightly copied database
* Authentication: Token-based

**[DIAGRAM PLACEHOLDER – Web Applications Overview]**

---

## **4. BioBridge / BioSync Service**

BioBridge (previously named BioSync) is a Windows Service installed on each client server. It is responsible for synchronizing data between Core and the web applications that require it.

| Product | BioBridge Usage | Sync Mode                       |
| ------- | --------------- | ------------------------------- |
| Portal  | ❌ Not Used      | Direct REST calls               |
| Insight | ✅ Required      | Hourly push from Core → Insight |
| Share   | ✅ Required      | Reads from nightly DB copy      |

**Notes from internal logs and team messages:**

* BioBridge runs hourly Insight syncs.
* Share depends entirely on the nightly database copy.
* Naming inconsistencies exist across documentation (“BioBridge,” “BioSync,” “Bridge Sync”).

---

## **5. Nightly Database Copy Process**

Share can only display **verified** and **approved** lab results. To ensure it never exposes unvalidated data:

1. A Windows Scheduled Task runs nightly at **1:00 AM**.
2. The task creates a database backup:

   ```
   Core_Copy.bak
   ```
3. BioBridge reads this copy and updates Share’s dataset.

### **Common operational issues**

* If disk space is low, the copy task fails (e.g., exit code 0x1).
* When the task fails, Share displays outdated information until the next successful run.
* Backup paths vary between environments, which introduces support challenges.

**[DIAGRAM PLACEHOLDER – Nightly Copy Flow]**

---

## **6. Hosting & Deployment Model**

Each client’s environment is self-contained and manually provisioned by the CS team. Although configurations are meant to be 90% identical across labs, variations exist due to licensing and legacy setups.

### **Standard deployment checklist:**

* Install SQL Server
* Deploy IIS sites:

  * Portal (port 8080)
  * Insight (port 8081)
  * Share (port 8082)
* Install and configure BioBridge/BioSync
* Configure Windows Scheduled Tasks for:

  * Nightly DB Copy (for Share)
  * Hourly Sync (for Insight)
* Validate authentication flows
* Test data movement end-to-end

---

## **7. Data Flow Summary**

Below is a simplified view of how data moves between components:

### **Portal → Core**

* Portal sends sequencing requests directly to Core through REST API calls.
* Real-time, two-way communication.

### **Core → Insight**

* BioBridge runs hourly to update Insight’s analytics tables.
* Good for near-real-time dashboards.

### **Core → Nightly Copy → Share**

* Nightly task creates Core_Copy.bak.
* BioBridge uses this file to refresh Share once per day.
* Ensures Share exposes only validated results.

**[DIAGRAM PLACEHOLDER – Full Data Flow Diagram]**

---

## **8. Known Documentation Gaps**

During review of internal communication and logs, several inconsistencies surfaced:

* Mixed terminology for the same service (BioBridge vs. BioSync vs. Bridge Sync).
* Missing diagrams showing how components interact.
* Unclear or inconsistent backup paths across different labs.
* No central reference explaining the purpose of the nightly copy.

This System Overview aims to standardize these definitions.

---

## **9. Assumptions**

To build a unified understanding, the following assumptions have been made:

* BioBridge and BioSync are interchangeable names for the same Windows Service.
* Share uses only the nightly copy and does not connect directly to Core.
* Authentication mechanisms listed represent the intended design; legacy exceptions may exist.
* Backup locations should be standardized, even though current deployments differ.

---

## **10. Conclusion**

The GeneSys ecosystem is a modular, single-tenant platform centered around GeneSys Core. Its surrounding web applications, BioBridge service, and scheduled tasks form a controlled workflow that ensures accuracy, data integrity, and compliance for laboratory clients.

This document provides a unified reference to help new Support, CS, and QA team members quickly understand how the system behaves. Future improvements should include standardized naming, consistent backup paths, environment-level diagrams, and updated operational runbooks.
