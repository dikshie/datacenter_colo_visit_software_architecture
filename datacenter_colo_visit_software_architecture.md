# Data Center Colocation Visit & Work Permit System

## High-Level Software Architecture Specification

---

### 1. Executive Summary

The Data Center Colocation Visit & Work Permit System is a dedicated operational intake and workflow automation platform. It manages the entire lifecycle of tenant physical access requests—from initial web intake through automated infrastructure mapping, Request Tracker (RT) integration, dual-audience notification dispatch, and physical single-page A4 document generation for on-site escort and custody tracking.

---

### 2. Core Business Invariants & Operational Rules

1. **The 1:1:1 Operational Principle:**
   $$\mathbf{1\text{ Scheduled Date} = 1\text{ RT Ticket Number} = 1\text{ Printed A4 Permit}}$$
   Each authorized visit date corresponds to an independent operational pass and Request Tracker record.

2. **Same-Date Deduplication & Multi-Sheet Dispatch:**
   * **Rule:** If a tenant submits a request for a visit date that already has an active RT ticket for their company, the system **reuses the existing RT Ticket Number** rather than spawning duplicates.
   * **Physical Handling:** Because subsequent submissions for the same date typically represent a different work crew, distinct vehicle set, or separate arrival time, the system produces an independent physical permit sheet (e.g., *Sheet 1 of 2*, *Sheet 2 of 2*) under the same RT ticket number.

3. **Operating Windows & Calendar Classification:**
   * **Standard Weekdays (Mon–Fri, non-holiday):** `16:30 – 19:30`
   * **Weekends & Official National Holidays:** `08:30 – 17:00`
   * The system automatically evaluates dates, identifies Japanese statutory/substitute holidays, assigns the correct window, and prints explicit cutoff warnings on the permit.

4. **Monthly Standing Access Pass (30-Day Exception):**
   * Submissions spanning a full 30-day (1 month) block generate **1 PARENT RT Ticket**.
   * When technicians physically arrive or call in advance, the on-duty engineer independently executes `rt-ticket-colo-maker` to spawn a **CHILD RT Ticket** for that specific day's work order and physical A4 permit.

---

### 3. System Architecture Diagram

```text
                     [ CLIENT / TENANT INTAKE ]  
                                  │  
                                  │ 1. Submits Visit Request  
                                  │    • Submitter (Name, Phone, Email) + Up to 5 CCs  
                                  │    • Dates (1 to 30 Days)  
                                  │    • Company ──► Rooms (Multi) ──► Locked Racks  
                                  │    • Vehicles (0 to 3 Plates / NONE)  
                                  │    • Visiting Roster (1 to 15 Technicians)  
                                  ▼  
 ┌─────────────────────────────────────────────────────────────────────────────┐  
 │                      APPLICATION & GATEWAY LAYER                            │  
 │  • Input Validation & Roster/Vehicle Boundary Checks                        │  
 │  • Operating Window Resolution (Weekday Evening vs. Weekend/Holiday Daytime)│  
 └──────────────────────┬──────────────────────────────────────────────────────┘  
                        │  
                        │ 2. Pre-Execution Database Check  
                        ▼  
 ┌─────────────────────────────────────────────────────────────────────────────┐  
 │              DATABASE LOOKUP & TICKET DEDUPLICATION ENGINE                  │  
 │                                                                             │  
 │  Does an active RT Ticket exist for [Company + Visit Date]?                 │  
 │     ├── [ YES ] ──► REUSE existing RT Ticket #; Increment Sheet #           │  
 │     └── [ NO  ] ──► Call `rt-ticket-colo-maker`                             │  
 │                        • Consecutive Dates ──► Single "one-shot" call       │  
 │                        • Non-Consecutive   ──► Day-by-day iteration         │  
 │                        • 30-Day Blanket    ──► Single PARENT ticket call    │  
 └──────────────────────┬──────────────────────────────────────────────────────┘  
                        │  
                        │ 3. Hydrate & Commit Records  
                        ▼  
 ┌─────────────────────────────────────────────────────────────────────────────┐  
 │                             PERSISTENCE LAYER                               │  
 │  • Master Submissions & CC Distribution Lists                               │  
 │  • Daily Visit Anchors (RT Ticket References)                               │  
 │  • Multi-Room & Rack Allocations                                            │  
 │  • Independent Permit Sheet Records & Technician Rosters                    │  
 └──────────────────────┬──────────────────────────────────────────────────────┘  
                        │  
                        │ 4. Trigger Dispatch  
                        ▼  
 ┌─────────────────────────────────────────────────────────────────────────────┐  
 │                 OPERATIONS & NOTIFICATION DISPATCH LAYER                    │  
 ├────────────────────────────────────────┬────────────────────────────────────┤  
 │  A. EXTERNAL CLIENT ACKNOWLEDGMENT     │  B. INTERNAL DC OPS / NOC ALERT    │  
 │  • Recipients: Submitter + Up to 5 CCs │  • Recipients: DC Ops & Guard Desk │  
 │  • Confirms intake receipt & RT ticket │  • Actionable summary metrics      │  
 │  • Outlines authorized access window   │  • Deep-links to print views       │  
 └────────────────────────────────────────┴────────────────────────────────────┘  
                                          │  
                                          │ 5. Security & Floor Execution  
                                          ▼  
 ┌─────────────────────────────────────────────────────────────────────────────┐  
 │                     PHYSICAL A4 DOCUMENT ENGINE                             │  
 │  • Rendered on single-page A4 portrait for clipboard use                    │  
 │  • Header: RT Ticket #, Sheet #, Submitter, Company, Hours, Rooms/Racks     │  
 │  • Vehicle Block: Registered Plates (1-3) or NONE                           │  
 │  • Roster Table: Up to 15 Technicians with Check-In & Check-Out times       │  
 │  • Check-In Escort: Room verification & Borrowed Tool list                  │  
 │  • Check-Out Escort: Rack locking, room clearance, & Tool return audit      │  
 └─────────────────────────────────────────────────────────────────────────────┘  
```

---

### 4. Detailed Component Specifications

#### A. Client Intake Layer

* **Submitter & Stakeholder Distribution:** Collects the submitter's full name, telephone number, and primary email, alongside up to 5 optional CC email addresses for supervisors, project managers, or subcontractors.  
* **Hierarchical Space Resolution:**  
  * Selecting a company queries the static mapping matrix.  
  * The form renders checkboxes for only authorized rooms leased by that client.  
  * Checking any room dynamically resolves and locks its associated rack numbers.  
* **Vehicle Registration:**  
  * Conditional toggle: Walking/Public Transit (`NONE`) vs. Driving.  
  * Driving option expands up to 3 optional license plate input fields.  
* **Dynamic Technician Roster:**  
  * Elastic table supporting 1 to 15 technician entries (`Full Name`, `Company/Employer`).  
* **Multi-Date Scheduling:**  
  * Accepts single dates, multi-day lists (consecutive or non-consecutive), or a 30-day blanket range.

#### B. Orchestration & Ticketing Bridge (`rt-ticket-colo-maker`)

* **Deduplication Check:** Before invoking the CLI tool, the backend checks if `(Company, Visit Date)` already has an assigned RT ticket. If so, it attaches the new submission to that ticket as an additional sheet.  
* **Consecutive Dates:** Dispatches date ranges in a single one-shot batch execution to `rt-ticket-colo-maker`.  
* **Non-Consecutive Dates:** Executes sequential day-by-day calls.  
* **30-Day Range:** Invokes parent-mode ticketing, generating a single PARENT RT Ticket covering the month.

#### C. Notification Dispatch Layer

* **External Client Confirmation:** Sends a single consolidated email to the submitter and all CCs listing the assigned RT Ticket Numbers, scheduled dates, and authorized time windows.  
* **Internal Operations Dispatch:** Alerts the DC NOC and facilities team with visitor counts, target rooms, vehicle registrations, and direct 1-click links to print work orders.

#### D. Physical Document Rendering Layer

* **Single-Page Form Factor:** Strict CSS `@media print` rules ensure the work order stays within 1 A4 portrait page, even with 15 roster rows.  
* **Custody & Escort Sign-Offs:**  
  * **Check-In Escort:** Captures duty engineer signature, entry timestamp, room initialing, and a dedicated field for borrowed DC tools (cage nut tools, console cables, fiber cleaners, torque screwdrivers).  
  * **Check-Out Escort:** Captures exit escort engineer signature, departure timestamp, work area inspection sign-off, and an explicit tool reconciliation check (`Returned in Good Condition` vs. `NOT RETURNED / Missing`).

---

### 5. Standardized Printable Work Order Layout

```text
┌────────────────────────────────────────────────────────────────────────────────────────┐  
│ DATA CENTER ON-SITE ACCESS PERMIT & WORK ORDER                                         │  
│ RT Ticket #: #128492  [ SHEET 1 OF 1 ]                     Submission: 2026-09-17 14:10│  
├────────────────────────────────────────────────────────────────────────────────────────┤  
│ SCHEDULED VISIT & AUTHORIZED OPERATING WINDOW                                          │  
│ Visit Date:        2026-10-01 (Thursday - Weekday)                                     │  
│ Permitted Window:  16:30 – 19:30 [ ACCESS STRICTLY DENIED OUTSIDE THIS WINDOW ]        │  
├────────────────────────────────────────────────────────────────────────────────────────┤  
│ SUBMISSION DETAILS                                                                     │  
│ Submitter: Taro Tanaka                     Company: Alpha Tech Capital                 │  
│ Contact:   +81-3-5555-0199                 Email:   t-tanaka@alphatech.example         │  
│ CC List:   lead@alphatech.example, vendor@fastfiber.example                            │  
├────────────────────────────────────────────────────────────────────────────────────────┤  
│ AUTHORIZED TARGET LOCATION(S)                                                          │  
│ Room Name                 │ Authorized Rack(s)                                         │  
│ ──────────────────────────┼─────────────────────────────────────────────────────────── │  
│ Server Hall 1A            │ Rack A-01, Rack A-02                                       │  
│ Server Hall 2B            │ Rack B-14                                                  │  
├────────────────────────────────────────────────────────────────────────────────────────┤  
│ AUTHORIZED VEHICLES                                                                    │  
│ [x] NONE (Transit / Walking)                                                           │  
│ [ ] Registered Plates: 1: _______________  2: _______________  3: _______________      │  
├────────────────────────────────────────────────────────────────────────────────────────┤  
│ AUTHORIZED PERSONNEL ROSTER (15 MAX)                                                   │  
│ #   | Technician Full Name     | Company / Employer      | Check-In  | Check-Out       │  
│ ────┼──────────────────────────┼─────────────────────────┼───────────┼─────────────────│  
│ 01  | Jane Doe                 | Alpha Tech Capital      | [       ] | [       ]       │  
│ 02  | Kenji Sato               | FastFiber Systems       | [       ] | [       ]       │  
│ ..  | ...                      | ...                     | [       ] | [       ]       │  
│ 15  | ________________________ | _______________________ | [       ] | [       ]       │  
├────────────────────────────────────────────────────────────────────────────────────────┤  
│ DUTY ENGINEER ESCORT SIGN-OFF                                                          │  
│                                                                                        │  
│ 1. CHECK-IN ESCORT (ENTRY TO ROOMS & TOOL SIGN-OUT)                                    │  
│    Escort Engineer: _________________________  Sign: _________________ Time: [ ____:____ ] │  
│    Rooms Escorted Into: [ ] Server Hall 1A    [ ] Server Hall 2B                       │  
│    Tools Borrowed:                                                                     │  
│    [ ] None                                                                            │  
│    [ ] Borrowed: _____________________________________________________________________ │  
│                                                                                        │  
│ 2. CHECK-OUT ESCORT (EXIT FROM ROOMS & TOOL RETURN)                                    │  
│    Escort Engineer: _________________________  Sign: _________________ Time: [ ____:____ ] │  
│    All Target Racks Locked & Rooms Cleared: [ ] OK                                     │  
│    Tool Return Status:                                                                 │  
│    [ ] N/A (No Tools Borrowed)                                                         │  
│    [ ] All Tools Returned in Good Condition                                            │  
│    [ ] NOT RETURNED / Missing: _______________________________________________________ │  
└────────────────────────────────────────────────────────────────────────────────────────┘  
```
