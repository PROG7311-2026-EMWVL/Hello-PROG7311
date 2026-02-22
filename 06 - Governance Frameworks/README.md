# 🚀 Enterprise Architecture Activity – Freelancer Subscription Upgrade

## 📝 Objective
Your task is to architect a modernised subscription system for Freelancer.dev using enterprise frameworks.

You must apply:
* TOGAF (ADM Phases A–D)
* Zachman Framework
* Optionally, ITIL

The goal is to design a scalable, governed, and operationally mature subscription system.

## 💡 Scenario

Freelancer.dev is a platform where:
* Freelancers advertise services
* Clients browse and book consultations
* Payments are processed via PayFast
* Authentication is handled by Firebase
* Emails are sent via SendGrid

The business is introducing a Subscription Tier System with the following requirements:
* Multiple tiers (Basic, Pro, Enterprise)
* Feature access controlled by subscription status
* Automatic expiry handling
* Failed payment management
* Auditability of all subscription changes
* Readiness for international scaling

You are hired as an enterprise consulting team to design the architecture and operational model.

## 🏗 PART 1 – TOGAF (Architecture Development Method)

Use ADM Phases A–D.
### 🔹 Phase A – Architecture Vision (10–15 minutes)

Define:
* The goal of the new subscription system.
* Key business and technical stakeholders.
* Risks if the subscription system is poorly implemented.
* 3 measurable success criteria.
* Keep answers concise and structured.

### 🔹 Phase B – Business Architecture (20 minutes)
#### 1️⃣ Define Key Roles
Identify the main roles involved in the subscription lifecycle (e.g., Subscriber, Admin, Finance, Support).

#### 2️⃣ Design the Subscription Lifecycle
Define at least five states, for example:
* Trial
* Active
* Suspended
* Expired
* Cancelled
Clearly describe what triggers movement between states.

#### 3️⃣ Define Business Rules

Create at least five enforcement rules, for example:
* Booking allowed only if subscription is Active.
* Account suspended after repeated failed payments.
* Expiry occurs automatically based on end date.
* Only Admin can override certain states.
* All state changes must be logged.
Be specific and logical.

### 🔹 Phase C – Information Systems Architecture (30 minutes)
#### A. Data Architecture
Define the core entities:
* User
* Subscription
* SubscriptionTier
* Payment
* Booking
* AuditLog

Draw a simple relationship diagram.

Then answer:
* Which data must be permanent?
* Which data must never be edited (audit purposes)?
* How do you preserve payment traceability?

#### B. Application Architecture

Design a service-oriented structure.

Break the system into logical services, for example:
* Authentication Service
* Subscription Service
* Booking Service
* Payment Service
* Notification Service

Then answer:
* Which service enforces subscription validation?
* Which service owns subscription status?
* How do services communicate?
* What happens if the payment service fails?

Be clear about responsibility boundaries.

### 🔹 Phase D – Technology Architecture (15 minutes)

Define:
* Deployment model (e.g., containerised services)
* Database structure (shared or per service)
* Logging and monitoring approach
* Integration with external systems
* Strategy for scaling to multiple countries

Keep this high-level and architectural.

## 🧱 PART 2 – ZACHMAN (Stakeholder Perspectives)

Choose the following roles:
* Planner
* Designer
* Developer

Complete the table for the Subscription System.

| Role      | What (Data) | How (Function) | Where (Network) | Who (People) | When (Time) | Why (Motivation) |
|-----------|-------------|----------------|-----------------|--------------|-------------|------------------|
| Planner   |             |                |                 |              |             |                  |
| Designer  |             |                |                 |              |             |                  |
| Developer |             |                |                 |              |             |                  |					

Keep each cell concise and role-specific.

Focus on how perspective changes across roles.

## 🧠 Reflection (10 minutes)
Answer briefly:
* What value did TOGAF bring to your thinking?
* How did Zachman differ from TOGAF?
* Which framework felt most natural to apply?
* What architectural decision would become most critical if the platform scaled globally?

## 📦 Submission Requirements
Submit a structured Markdown or PDF report containing:
* TOGAF breakdown (Phases A–D)
* Zachman table
* Reflection answers
* Diagrams where appropriate