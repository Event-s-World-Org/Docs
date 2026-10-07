# 1. Introduction

### 1.1 Purpose

The purpose of this document is to define in detail the functional and non-functional requirements for "EventsWorld," a comprehensive web platform focused on the efficient management of events.

### 1.2 Product Scope

"Event's World" is a multi-user platform. It centralizes event management, registration, and full logistics management for all types of events. It includes a dynamic registration flow to events and events management — all built on a robust Object-Oriented architecture and relational database.

---

## 2. Overall Description

### 2.1 User Roles and Permissions

The system empowers all users under a Role-Based Access Control (RBAC) scheme, with specific views and hierarchies:

---

1. **Super Admin (Core Team):**

* *Access:* Global, platform-wide administrative view.
* *Capabilities:* Event management and user auditing.

---

2. **Organizer (Event Creator):**

* *Access:* Internal management view for events they have created, of course. They has to be already logged in.
* *Capabilities:* Full control over logistics and definition of additional required fields. Has exclusive access to the **Event Balance** and is the main role authorized to manage their own events .

---

3. **Co-Administrator (Guest):** (not available - SOON)

* *Access:* Internal management view for events they were invited to.
* *Capabilities:* Supports logistics, scheduling, and team organization. **Strict restriction:** No access to the “Event Balance”.

---

4. **Regular User (Participant / Guest):**

* *Access:* Public browsing view and public event registration forms.
* *Capabilities:* Can search for events and enroll directly via forms without needing to create a platform account. Account creation is mandatory only for Event Creators (Organizers).


# Functional Requirements (FR)

## Module 1: Authentication, Registration, and Profile

* **FR-01.1 (Base Registration):** Platform accounts are only required for Organizers. They will be created by requiring: Email, Password, Full Name, and Date of Birth. Guests can enroll in events without registering to the site.
* **FR-01.2 (Identity and Access Management - IAM):** Authentication and Authorization will be managed using **Supabase Auth**. This will handle user registration, secure credential storage, and token generation (JWT).
* **FR-01.3 (Security):** Secure communication via JWTs provided by Supabase, ensuring Role-Based Access Control (RBAC) validations across the platform.

---

## Module 2: Event Creation and Configuration (Organizer)

* **FR-02.1 (Creation and Location):** Events must be created by specifying Country, City, dates, title, description, maxIncriptionDate, and max of attendees.

* **FR-02.2 (Payment Configuration):** During event creation, the Organizer MUST upload their payment QR code. They must also define via a toggle if "Payment Plans" (e.g., monthly partial payments) will be allowed for this event.

* **FR-02.3 (Custom Extra Fields):** The Organizer will define the required fields for the event's dynamic registration form. A mandatory field that the system will always enforce for ALL remote event forms is the "Payment Voucher Upload".

---

## Module 3: Event Logistics Tools

* **FR-03.1 (Team Splitter):** Tools to distribute participants into groups.

* **FR-03.2 (Logistics Filtering by Plan):** When using the "Team Splitter" or reviewing participant lists, the Organizer can filter participants based on their payment state (completed - partial)

* **FR-03.3 (Remote Registration & Payment):** Guests enroll via a dynamic form generated based on the organizer's requirements. This form will automatically display the Organizer's QR code and MUST include a mandatory "Payment Voucher Upload" field. If the Organizer enabled "Payment Plans" for this event, the guest can select to pay in full or enter a partial amount to start a payment plan.

* **FR-03.4 (Manual/In-Person Registration & Payment):** The Organizer can manually register an attendee in person by filling out the form on their behalf. The Organizer selects the payment method:
  * **QR:** The Organizer's device displays their QR code for the attendee to scan and pay on the spot, followed by pressing a "Confirm Payment" button.
  * **Cash:** The Organizer registers the exact amount received (full or partial).
  * If the event has "Payment Plans" enabled, the Organizer can register partial cash/QR amounts to start a payment plan for the user.

* **FR-03.5 (Remote Payment Review Page):** Organizers will have a dedicated Review Page listing pending/paid users from remote forms. It will display a large, legible image of the uploaded payment voucher, along with action buttons to: "Accept Payment", "Put in Observation", or "Mark as Error".

* **FR-03.6 (Event Dashboard & Financial Metrics):** Each event will have a dashboard displaying the list of enrolled users, with filters for their payment state (e.g., fully paid vs. on a payment plan). It will also show financial metrics comparing "Real Money" (actual confirmed payments in the ledger) versus "Estimated Money" (expected total based on full ticket prices and payment plans).
* **FR-03.7 (Change user data):** As there could be user with partial payments, and sometimes incorrect data by part of the forms, the administrator has to be able to edit this data with enough warnings of it.



# Non-Functional Requirements (NFR)

## NFR-01: User Interface, Experience, and Content Delivery (UI/UX & CDN)

* **NFR-01.1 (Aesthetics and Hybrid CSS Approach):** The frontend will be developed in **Angular** using **TypeScript**, with a hybrid styling strategy:

  * **Tailwind CSS:** It will be used strictly for overall layout, grid system (Mobile First), backgrounds, typography, and to achieve the "Glassmorphism" visual pattern (translucent elements with background blur and subtle borders) in cards and main containers.
  * **Angular Material:** It will be integrated exclusively to accelerate the development of complex functional components requiring high interactivity and accessibility, such as: MatDatepicker (event date selection), MatSelect/MatSlideToggle (extra field configuration), and MatDialog (modals for QR upload).

* **NFR-01.2 (Responsive Design):** Mandatory "Mobile First" approach, assuming heavy mobile usage, with native Dark Mode support.

* **NFR-01.3 (CDN & Media Storage):** **Cloudflare R2** will be used as the Content Delivery Network (CDN) and object storage solution for hosting user-uploaded media (such as large, legible payment vouchers) and event assets.

---

## NFR-02: Performance and Architecture (Backend)

* **NFR-02.1 (Structural Framework):** The backend will be built using **C#** and **.NET (ASP.NET Core Web API)**.

* **NFR-02.2 (Object-Oriented Paradigm - OOP):** Backend development will follow SOLID principles. Classes, Interfaces, and Inheritance will be used.

* **NFR-02.3 (Native Dependency Injection):** The native .NET Dependency Injection container (`IServiceCollection`) will be exclusively used to decouple Controllers, Services, and Repositories.

* **NFR-02.4 (ORM & Data Access):** **Entity Framework Core (EF Core)** with the `Npgsql` provider will be used for database access and migrations to PostgreSQL, supporting the Ledger pattern and JSONB operations.

---

## NFR-03: Data Modeling and Ledger Pattern (PostgreSQL)

* **NFR-03.1 (Relational Database):** **PostgreSQL** will be the primary database engine. `JSONB` columns will be used to store dynamic user responses for "Extra Fields."

* **NFR-03.2 (Strict Ledger Pattern Implementation):** The use of static and mutable `balance` fields (e.g., a simple `UPDATE balance = balance - X`) is strictly prohibited. The architecture must include at least three immutable entities:

  * `Accounts` (Financial accounts for Users, Events, and the Platform itself)
  * `Transactions` (Logical grouping of an operation, e.g., "Event X registration")
  * `Entries` (Individual transaction movements, recording positive and negative amounts)

* **NFR-03.3 (Balance Calculation):** The available balance for any actor will be computed dynamically (or via materialized views) by summing all confirmed `Entries`.

* **NFR-03.4 (Ledger Reversals):** To handle refunds, cancellations, or corrections, modifying or deleting existing entries is strictly prohibited. The system must implement a "Reversal" mechanism where a new `Transaction` is created containing inverse `Entries` (mathematically negating the original amounts). This reversal transaction must be linked to the original transaction via a `reversed_transaction_id` reference.
