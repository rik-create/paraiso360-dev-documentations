# Paraiso360 Rules, Workflows & Policies.

## Table of Contents

1. [System Overview](#system-overview)
2. [Lot Classifications](#lot-classifications)
3. [Lot Status Progression](#lot-status-progression)
4. [Purchase Types](#purchase-types)
5. [Payment Methods & Verification](#payment-methods--verification)
6. [Warning Level System](#warning-level-system)
7. [Penalty Calculation](#penalty-calculation)
8. [Forfeiture Process](#forfeiture-process)
9. [Component Workflows](#component-workflows)
10. [Configuration Settings](#configuration-settings)

---

## 1. System Overview

The Cemetery Lot Management System handles the complete lifecycle of cemetery lot sales, from reservation through payment completion or forfeiture. The system differentiates between **Pre-Need** (advance planning) and **At-Need** (immediate burial requirement) purchases, each with distinct rules and pricing.

**Key Technologies:**

- **Monitoring System:** Supabase Cron Jobs + Edge Functions
- **Automated Processing:** Warning level progression, penalty calculations, timeout management

---

## 2. Lot Classifications

All lot classifications include configurable internment capacity (e.g., 4 skeletal, 2 fresh).

| Classification    | Base Price  | Internment Capacity                      |
| ----------------- | ----------- | ---------------------------------------- |
| **Diamond**       | ₱150,000.00 | Configurable (e.g., 4 skeletal, 2 fresh) |
| **Platinum**      | ₱120,000.00 | Configurable (e.g., 4 skeletal, 2 fresh) |
| **Family Estate** | ₱300,000.00 | Configurable (e.g., 4 skeletal, 2 fresh) |
| **Gold**          | ₱75,000.00  | Configurable (e.g., 4 skeletal, 2 fresh) |

**Note:** Additional classifications can be added through the admin panel. Both internment capacity numbers are configurable per classification.

---

## 3. Lot Status Progression

The system follows a strict status workflow:

```
Available → Reserved → Partial/Sold
```

### Status Definitions

**Available**

- Lot is open for reservation
- No current owner or pending transactions

**Reserved**

- Triggered when reservation modal is opened and lot is selected
- Duration: 30 minutes (configurable)
- Automatically returns to Available if:
  - Timer expires without payment
  - Modal is closed without completing transaction

**Partial**

- Payment has been initiated but not completed in full
- Lot is owned but has outstanding balance
- Subject to warning level system

**Sold**

- Full payment received and verified
- Lot ownership transferred completely
- No further payment required

---

## 4. Purchase Types

### 4.1 Pre-Need Purchase

**Definition:** Advance planning for future burial needs

**Key Features:**

- Installment plans allowed
- Spot cash discount opportunities
- 6-level warning system for overdue payments
- Standard base pricing
- 15% minimum down payment (configurable)

#### Pre-Need Payment Options

**A. Spot Cash (Full Payment with Discount)**

Discounts based on payment timeline:

| Payment Within | Discount | Example (₱100,000 lot) |
| -------------- | -------- | ---------------------- |
| 7 days         | 10% off  | ₱90,000                |
| 15 days        | 7% off   | ₱93,000                |
| 30 days        | 5% off   | ₱95,000                |

**Schedule Generation:** Aligned to selected discount period (7, 15, or 30 days)

**Example:**

- Customer reserves a ₱100,000 Gold lot
- Chooses 7-day spot cash option
- Final price: ₱90,000
- Must pay within 7 days to receive discount
- If paid within 7 days: Status → Sold

**B. Standard Installment**

Available terms: 12, 24, or 36 months

**Example - 24-month installment:**

- Lot price: ₱120,000 (Platinum)
- Minimum down payment (15%): ₱18,000
- Remaining balance: ₱102,000
- Monthly payment: ₱102,000 ÷ 24 = ₱4,250
- Schedule: Monthly payments aligned to reservation date

**C. Custom Installment**

Flexible payment terms (e.g., 2, 3, 5, 7, 8 months, etc.)

**Example - 10-month custom plan:**

- Lot price: ₱75,000 (Gold)
- Down payment (20%): ₱15,000
- Remaining balance: ₱60,000
- Monthly payment: ₱60,000 ÷ 10 = ₱6,000
- Schedule: Custom-aligned monthly payments

---

### 4.2 At-Need Purchase

**Definition:** Immediate purchase for urgent burial requirements

**Key Features:**

- **1.5x base price multiplier**
- 75% minimum down payment (configurable)
- 72-hour payment deadline (configurable)
- No installment plans
- No spot cash discounts
- Special Level 7 warning for overdue

**Pricing Example:**

- Diamond lot base price: ₱150,000
- At-Need price: ₱150,000 × 1.5 = ₱225,000
- Minimum down payment (75%): ₱168,750
- Maximum remaining balance: ₱56,250
- Due within: 72 hours

**Schedule Generation:** Single due date set to 72 hours (configurable) from reservation

**Example Scenario:**

- Family needs immediate burial
- Selects Platinum lot (base: ₱120,000)
- At-Need price: ₱180,000
- Pays ₱135,000 down payment (75%)
- Remaining ₱45,000 due in 72 hours
- If not paid within 72 hours → Level 7 Manual Review

---

## 5. Payment Methods & Verification

### 5.1 Payment Methods

**Cash**

- Physical currency payment
- Automatic verification upon receipt
- No admin approval required

**Non-Cash**

- GCash, bank transfer, check, etc.
- Requires admin verification
- Status: "Pending Verification" until approved

### 5.2 Payment Status Workflow (PAY-002)

```
Non-Cash: Pending Verification → [Admin Review] → Verified/Failed
Cash: Automatic → Verified
```

### 5.3 Status Determination Logic

**When Full Payment + Cash:**

- Lot Status: **Sold**
- Payment Status: **Verified**

**When Partial Payment + Cash:**

- Lot Status: **Partial**
- Payment Status: **Verified**
- Enters warning level system

**When Any Amount + Non-Cash:**

- Lot Status: **Partial**
- Payment Status: **Pending Verification**
- Awaits admin approval
- If approved → Verified → May enter warning system if not full payment
- If rejected → Failed

**Example:**

1. Customer pays ₱50,000 down payment via GCash
2. Status becomes: Partial + Pending Verification
3. Admin reviews transaction proof
4. Admin approves → Status: Partial + Verified
5. Warning level system begins monitoring due dates

---

## 6. Warning Level System

### 6.1 Pre-Need Warning Levels (LOT-005)

The system automatically tracks payment due dates and escalates warning levels:

| Level          | Status              | Days Overdue | Action        | Penalty |
| -------------- | ------------------- | ------------ | ------------- | ------- |
| 🟢 **Level 1** | Active              | ≤0 days      | Normal        | None    |
| 🟡 **Level 2** | Grace Period        | 1-7 days     | Reminder      | None    |
| 🟠 **Level 3** | Overdue             | 8-29 days    | Warning       | Applied |
| 🔵 **Level 4** | First Warning       | 30-59 days   | Urgent notice | Applied |
| 🔴 **Level 5** | Final Warning       | 60-89 days   | Final notice  | Applied |
| ⚫ **Level 6**  | Forfeiture Eligible | 90+ days     | Admin review  | Applied |

**Grace Period Details (PAY-005):**

- Duration: 7 days after due date
- No penalties applied during grace period
- Customer receives gentle reminders
- Only applies to Pre-Need installment plans

**Example Timeline:**

- Due date: October 25
- Oct 26-Nov 1 (Days 1-7): Grace Period (Level 2) - No penalty
- Nov 2-Nov 22 (Days 8-29): Overdue (Level 3) - Penalty starts
- Nov 23-Dec 23 (Days 30-59): First Warning (Level 4) - Penalty continues
- Dec 24-Jan 22 (Days 60-89): Final Warning (Level 5) - Penalty continues
- Jan 23+ (Day 90+): Forfeiture Eligible (Level 6) - Admin decision required

---

### 6.2 At-Need Warning Level

| Level          | Status        | Condition         | Action                                                   |
| -------------- | ------------- | ----------------- | -------------------------------------------------------- |
| 🟣 **Level 7** | Manual Review | 72+ hours overdue | Payment processing BLOCKED - Requires admin intervention |

**Key Differences:**

- Only applies to At-Need purchases
- No grace period
- No installment options
- Immediate admin review required after 72-hour deadline
- Payment processing is blocked until admin manually reviews and decides next steps

**Example:**

- At-Need lot reserved: Monday 10:00 AM
- 72-hour deadline: Thursday 10:00 AM
- If unpaid by Thursday 10:01 AM → Level 7
- System blocks all payment attempts
- Admin must manually review and either:
  - Extend deadline
  - Proceed with forfeiture
  - Allow immediate full payment

---

## 7. Penalty Calculation (PEN-002, PAY-005)

### 7.1 Formula

```
Penalty = Monthly Payment × 2% × Penalty Months
```

**Configuration:**

- Default rate: 2% monthly (configurable)
- Grace period: 7 days (no penalties)
- Only applies to Pre-Need purchases

### 7.2 Calculation Example

**Scenario:**

- Monthly payment: ₱5,000
- Due date: October 25
- Current date: November 30 (36 days overdue)

**Step 1: Determine penalty period**

- Days overdue: 36 days
- Grace period: 7 days (no penalty)
- Penalty days: 36 - 7 = 29 days
- Penalty months: 29 days ÷ 30 = 0.97 months

**Step 2: Calculate penalty**

- Penalty = ₱5,000 × 2% × 0.97
- Penalty = ₱5,000 × 0.02 × 0.97
- **Penalty = ₱97.00**

**Step 3: Total amount due**

- Monthly payment: ₱5,000.00
- Penalty: ₱97.00
- **Total due: ₱5,097.00**

### 7.3 Penalty Display in Payment Processing

**Warning Levels Showing Penalty:**

- Level 3: Overdue (8-29 days) → Show penalty field
- Level 4: First Warning (30-59 days) → Show penalty field
- Level 5: Final Warning (60-89 days) → Show penalty field
- Level 6: Forfeiture Eligible (90+ days) → Show penalty field

**Warning Levels WITHOUT Penalty:**

- Level 1: Active → No penalty field
- Level 2: Grace Period → No penalty field

**At-Need Status:**

- Level 7: Manual Review → Payment processing BLOCKED

---

## 8. Forfeiture Process (FOR-002, FOR-003, FOR-004)

### 8.1 Eligibility Check

**Automatic trigger:** Lot reaches Level 6 (90+ days overdue)

**System actions:**

1. Flags lot for forfeiture review
2. Sends notification to admin dashboard
3. Blocks customer-initiated payments
4. Generates forfeiture recommendation report

### 8.2 Admin Decision Process

**Option A: Proceed with Forfeiture**

- All verified payments retained by cemetery
- Outstanding balance written off
- Customer loses all rights to lot
- Lot status reset to "Available"
- Warning level reset to Level 1
- Payment schedules cancelled
- Ownership records cleared
- Financial data removed from active lot record

**Option B: Allow Immediate Payment**

- Admin grants one-time payment extension
- Customer must pay immediately
- Full outstanding balance + all penalties due
- No further extensions allowed
- If payment received → Lot status updated accordingly
- If payment not received → Automatic forfeiture

### 8.3 Post-Forfeiture Lot Status

**Complete system reset:**

- Lot Status: Available
- Warning Level: 1 (Active)
- Owner: None
- Payment History: Archived (not deleted)
- Balance Due: ₱0.00
- Lot becomes available for new reservations

**Example:**

- Original purchase: ₱120,000 Platinum lot
- Paid to date: ₱45,000 (verified)
- Outstanding balance: ₱75,000
- After forfeiture:
  - Cemetery retains: ₱45,000
  - Customer loses: ₱45,000 paid + lot ownership
  - Outstanding ₱75,000 written off
  - Lot returned to sales inventory

---

## 9. Component Workflows

### 9.1 ReserveLot Component (LOT-004)

**Purpose:** Handle lot reservation and initial payment configuration

**Requirements:**

- Only "Available" lots can be reserved
- 30-minute timeout (configurable)
- Timeout resets if modal closed without completion

**Workflow Steps:**

**Step 1: Lot Selection**

- User browses available lots
- Clicks "Reserve" on desired lot
- System checks lot status
- If Available → Open reservation modal
- If not Available → Show error message

**Step 2: Purchase Type Selection**

- User chooses: Pre-Need or At-Need
- System adjusts pricing accordingly
- At-Need: Applies 1.5x multiplier
- Pre-Need: Shows base pricing

**Step 3: Payment Plan Configuration**

*For Pre-Need:*

- Spot Cash: Select discount tier (7/15/30 days)
- Standard Installment: Choose 12/24/36 months
- Custom Installment: Enter custom duration
- System calculates minimum down payment (15%)
- System generates payment schedule

*For At-Need:*

- No plan options
- System sets 72-hour deadline
- Enforces 75% minimum down payment
- Generates single-payment schedule

**Step 4: Initial Payment**

- User enters payment amount (must meet minimum)
- Selects payment method (Cash/Non-Cash)
- Submits payment
- System determines final status (Sold/Partial)
- System determines payment verification status

**Step 5: Confirmation**

- Display payment receipt
- Show payment schedule
- Provide lot ownership details
- Email confirmation sent

---

### 9.2 PaymentProcessing Component

**Purpose:** Process subsequent payments for Partial status lots

**Search Functionality:**

- Search by lot number, owner name, or contact information
- Filter by warning level
- Filter by purchase type (Pre-Need/At-Need)
- Display outstanding balance

**Warning Level Display Logic:**

**Pre-Need Lots:**

*Level 1 (Active):*

- Show: Next due date, amount due
- Hide: Penalty fields
- Allow: Normal payment processing

*Level 2 (Grace Period):*

- Show: Days in grace period, next due date
- Hide: Penalty fields
- Display: Friendly reminder message
- Allow: Normal payment processing

*Level 3-5 (Overdue, First Warning, Final Warning):*

- Show: Days overdue, penalty amount, total due
- Display: Penalty breakdown
- Highlight: Urgency level
- Allow: Payment with penalty included

*Level 6 (Forfeiture Eligible):*

- Show: Forfeiture warning, total penalties
- Display: Admin contact information
- Block: Standard payment processing
- Note: "Admin approval required for payment"

**At-Need Lots:**

*Level 7 (Manual Review):*

- Display: "PAYMENT BLOCKED"
- Show: "Contact administrator immediately"
- Block: All payment processing
- Display: Admin contact information
- Note: Manual intervention required

**Payment Processing Steps:**

1. Verify lot and customer information
2. Display current balance and any penalties
3. Allow payment amount entry (can be partial or full)
4. Select payment method
5. Process payment
6. Update lot status if fully paid
7. Recalculate warning level
8. Generate payment receipt

---

## 10. Configuration Settings

All system parameters are configurable through admin panel:

### 10.1 Global Settings

| Setting             | Default Value | Description                      |
| ------------------- | ------------- | -------------------------------- |
| Reservation Timeout | 30 minutes    | Auto-release reserved lots       |
| Grace Period        | 7 days        | No-penalty period after due date |
| Penalty Rate        | 2% monthly    | Monthly penalty percentage       |

### 10.2 Pre-Need Settings

| Setting                   | Default Value | Description                                |
| ------------------------- | ------------- | ------------------------------------------ |
| Minimum Down Payment      | 15%           | Minimum initial payment                    |
| Spot Cash 7-day Discount  | 10%           | Full payment within 7 days                 |
| Spot Cash 15-day Discount | 7%            | Full payment within 15 days                |
| Spot Cash 30-day Discount | 5%            | Full payment within 30 days                |
| Forfeiture Threshold      | 90 days       | Days overdue before forfeiture eligibility |

### 10.3 At-Need Settings

| Setting               | Default Value | Description              |
| --------------------- | ------------- | ------------------------ |
| Price Multiplier      | 1.5x          | Urgency pricing factor   |
| Minimum Down Payment  | 75%           | Minimum initial payment  |
| Payment Deadline      | 72 hours      | Time to complete payment |
| Manual Review Trigger | 72 hours      | When Level 7 activates   |

### 10.4 Lot Classification Settings

Each classification can configure:

- Base price
- Internment capacity (skeletal)
- Internment capacity (fresh)
- Description and features

**Adding New Classifications:**

1. Access admin panel
2. Navigate to "Lot Classifications"
3. Click "Add New Classification"
4. Enter classification name
5. Set base price
6. Configure internment capacities
7. Save and publish

---

## Appendix: Quick Reference Tables

### Status Transition Matrix

| Current Status | Cash Full Payment | Cash Partial         | Non-Cash Full       | Non-Cash Partial    |
| -------------- | ----------------- | -------------------- | ------------------- | ------------------- |
| Available      | → Sold (Verified) | → Partial (Verified) | → Partial (Pending) | → Partial (Pending) |
| Reserved       | → Sold (Verified) | → Partial (Verified) | → Partial (Pending) | → Partial (Pending) |
| Partial        | → Sold (Verified) | → Partial (Verified) | → Partial (Pending) | → Partial (Pending) |

### Payment Method Verification

| Method         | Verification | Admin Approval Required |
| -------------- | ------------ | ----------------------- |
| Cash           | Automatic    | No                      |
| GCash          | Manual       | Yes                     |
| Bank Transfer  | Manual       | Yes                     |
| Check          | Manual       | Yes                     |
| Other Non-Cash | Manual       | Yes                     |



---