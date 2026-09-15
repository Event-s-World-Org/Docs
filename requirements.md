# 1. Introduction

### 1.1 Purpose

The purpose of this document is to define in detail the functional and non-functional requirements for "EventsWorld," a comprehensive web platform focused on the efficient management of events.

### 1.2 Product Scope

"Event's World" is a multi-user platform. It centralizes event management, registration, and full logistics management for all types of events. It includes a technical support module, dynamic registration flow to events and events management — all built on a robust Object-Oriented architecture and relational database.

---

## 2. Overall Description

### 2.1 User Roles and Permissions

The system empowers all users under a Role-Based Access Control (RBAC) scheme, with specific views and hierarchies:

---

1. **Super Admin (Core Team):**

* *Access:* Global, platform-wide administrative view.
* *Capabilities:* Event management, user auditing and Support Inbox administration.

---

2. **Organizer (Event Creator):**

* *Access:* Internal management view for events they have created.
* *Capabilities:* Full control over logistics and definition of additional required fields. Has exclusive access to the **Event Balance** and is the main role authorized to manage their own events .

---

3. **Co-Administrator (Guest):** (not available - SOON)

* *Access:* Internal management view for events they were invited to.
* *Capabilities:* Supports logistics, scheduling, and team organization. **Strict restriction:** No access to the “Event Balance”.

---

4. **Regular User (Participant):**

* *Access:* Public browsing view and “My Events” dashboard.
* *Capabilities:* Event search, registration and own account administration


# Functional Requirements (FR)

## Module 1: Authentication, Registration, and Profile

* **FR-01.1 (Base Registration):** The main account on the platform will be created by requiring: Email, Password, Full Name, Date of Birth, and a unique Username.
* **FR-01.2 (Security):** Authentication via JSON Web Tokens (JWT). Passwords must be encrypted using strong algorithms (`bcrypt` with a minimum of 10 rounds).

---

## Module 2: Event Creation and Configuration (Organizer)

* **FR-02.1 (Creation and Location):** Events must be created by specifying Country, City, dates, title, description and max of attendees.

* **FR-02.2 (Custom Extra Fields):** The Organizer can enable toggles to require specific data during registration (e.g., Blood Type, ID Number).

---

## Module 3: Event Logistics Tools

* **FR-03.1 (Team Splitter):** Tools to distribute participants into groups.

* **FR-03.2 (Logistics Filtering by Plan):** When using the "Team Splitter" or reviewing participant lists, the Organizer can filter participants based on their payment state (completed - partial)

* **FR-03.3 (Forms Creation and Management):** With data events and specified by the Organizer, the application is available to create two kind of forms: *Manual*, where the organizer registers a new attendee manually, with the option to make partial payments. Remote, where as a common online form, interested people can join to the event by registering themselves but with the only option of full payment (mandatory voucher).

* **FR-03.4 (Join Event Requests):** At the moment to send a Remote form, a Join event request would be created on administrator view, where all request details will be shown (specially to verify voucher on the admins/organization bank account)

* **FR-03.5 (Change user data):** As there could be user with partial payments, and sometimes incorrect data by part of the forms, the administrator has to be able to edit this data with enough warnings of it.



# Non-Functional Requirements (NFR)

## NFR-01: User Interface and User Experience (UI/UX)

* **NFR-01.1 (Aesthetics and Hybrid CSS Approach):** The frontend will be developed in *Angular*, using a hybrid styling strategy:

  * **Tailwind CSS:** It will be used strictly for overall layout, grid system (Mobile First), backgrounds, typography, and to achieve the "Glassmorphism" visual pattern (translucent elements with background blur and subtle borders) in cards and main containers.
  * **Angular Material:** It will be integrated exclusively to accelerate the development of complex functional components requiring high interactivity and accessibility, such as: MatDatepicker (event date selection), MatSelect/MatSlideToggle (extra field configuration), and MatDialog (modals for QR upload).

* **NFR-01.2 (Responsive Design):** Mandatory "Mobile First" approach, assuming heavy mobile usage, with native Dark Mode support.

---

## NFR-02: Performance and Architecture (Backend)

* **NFR-02.1 (Structural Framework):** The backend will be built using **NestJS (Node.js)** with **TypeScript**.

* **NFR-02.2 (Object-Oriented Paradigm - OOP):** Backend development will follow SOLID principles. Classes, Interfaces, and Inheritance will be used.

* **NFR-02.3 (Native Dependency Injection):** The native NestJS DI container (`@Injectable()`) will be exclusively used to decouple Controllers, Services, and Repositories.

---

## NFR-03: Data Modeling and Ledger Pattern (PostgreSQL)

* **NFR-03.1 (Relational Database):** **PostgreSQL** will be the primary database engine. `JSONB` columns will be used to store dynamic user responses for "Extra Fields."

* **NFR-03.2 (Strict Ledger Pattern Implementation):** The use of static and mutable `balance` fields (e.g., a simple `UPDATE balance = balance - X`) is strictly prohibited. The architecture must include at least three immutable entities:

  * `Accounts` (Financial accounts for Users, Events, and the Platform itself)
  * `Transactions` (Logical grouping of an operation, e.g., "Event X registration")
  * `Entries` (Individual transaction movements, recording positive and negative amounts)

* **NFR-03.3 (Balance Calculation):** The available balance for any actor will be computed dynamically (or via materialized views) by summing all confirmed `Entries`.
