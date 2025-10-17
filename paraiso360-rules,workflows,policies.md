# Cemetery Lot Management System - Comprehensive Guide

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

| Classification | Base Price | Internment Capacity |
|---------------|------------|---------------------|
| **Diamond** | ₱150,000.00 | Configurable (e.g., 4 skeletal, 2 fresh) |
| **Platinum** | ₱120,000.00 | Configurable (e.g., 4 skeletal, 2 fresh) |
| **Family Estate** | ₱300,000.00 | Configurable (e.g., 4 skeletal, 2 fresh) |
| **Gold** | ₱75,000.00 | Configurable (e.g., 4 skeletal, 2 fresh) |

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
|---------------|----------|----------------------|
| 7 days | 10% off | ₱90,000 |
| 15 days | 7% off | ₱93,000 |
| 30 days | 5% off | ₱95,000 |

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

| Level | Status | Days Overdue | Action | Penalty |
|-------|--------|--------------|--------|---------|
| 🟢 **Level 1** | Active | ≤0 days | Normal | None |
| 🟡 **Level 2** | Grace Period | 1-7 days | Reminder | None |
| 🟠 **Level 3** | Overdue | 8-29 days | Warning | Applied |
| 🔵 **Level 4** | First Warning | 30-59 days | Urgent notice | Applied |
| 🔴 **Level 5** | Final Warning | 60-89 days | Final notice | Applied |
| ⚫ **Level 6** | Forfeiture Eligible | 90+ days | Admin review | Applied |

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

| Level | Status | Condition | Action |
|-------|--------|-----------|--------|
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

## 9. Payment Reversal System

### 9.1 Overview

The payment reversal system handles rare cases where payments need to be undone due to errors, disputes, or special circumstances. The system maintains strict rules to ensure data integrity and prevent cascading issues.

**Important:** Payment reversals are exceptional operations and should only be performed after thorough verification.

---

### 9.2 Reversal Eligibility Requirements

**Core Requirements:**

**Payment Status Check**
- Only "Verified" payments can be reversed
- Payments marked as "Pending Verification" cannot be reversed
- Previously reversed payments cannot be reversed again
- Failed payments have no reversal option

**Dependency Order Enforcement**
- Must reverse payments in reverse chronological order
- Newer payments must be reversed before older ones
- System validates payment sequence before allowing reversal

**Example of Dependency Order:**
```
Payment History:
1. Oct 1 - Down Payment: ₱15,000 (Verified)
2. Oct 15 - Installment #1: ₱5,000 (Verified)
3. Nov 1 - Installment #2: ₱5,000 (Verified)

Reversal Order Required:
Step 1: Reverse Nov 1 payment first
Step 2: Then reverse Oct 15 payment
Step 3: Finally, down payment can be reversed

❌ BLOCKED: Cannot reverse down payment while installments exist
```

**Schedule Protection Rule**
- Down payments cannot be reversed if any schedule payments exist
- All scheduled payments must be reversed first
- This prevents orphaned payment schedules

---

### 9.3 Payment Type Detection & Rules

The system automatically detects payment type and applies corresponding rules:

| Payment Type | Detection Method | Key Characteristics |
|--------------|------------------|---------------------|
| **Spot Cash** | First payment = Full base price | Single payment, no schedules |
| **Down Payment** | First payment < Full price | Creates payment schedule |
| **Installment** | Subsequent scheduled payment | Part of payment schedule |
| **At-Need Initial** | First At-Need payment | 75%+ initial payment |
| **At-Need Remaining** | Second At-Need payment | Remaining balance payment |

---

### 9.4 Payment Type-Specific Reversal Behavior

#### A. Spot Cash Reversal

**Identification:**
- First payment in transaction history
- Payment amount = Full lot price (after discount)
- No payment schedule generated
- Lot status = "Sold"

**Reversal Actions:**
1. Remove payment record and all documents
2. Reset lot status to "Available"
3. Clear ownership information
4. Remove discount application record
5. Reset warning level to N/A
6. Clear all financial data

**Example:**
```
Original Transaction:
- Gold lot: ₱75,000 base price
- 7-day spot cash discount: 10% off
- Paid: ₱67,500 (cash, verified)
- Status: Sold

After Reversal:
- Lot Status: Available
- Owner: None
- Balance: ₱0
- Payment History: Cleared
- Available for new reservation
```

---

#### B. Down Payment Reversal

**Identification:**
- First payment in transaction history
- Payment amount < Full lot price
- Payment schedule exists
- Lot status = "Partial"

**Protection Check:**
```
BEFORE allowing reversal, system checks:
- Are there any subsequent payments? 
  → YES: Block reversal, show error
  → NO: Proceed with reversal
```

**Reversal Actions:**
1. Delete entire payment schedule
2. Remove down payment record and documents
3. Reset lot status to "Available"
4. Clear ownership information
5. Reset warning level to N/A
6. Clear all financial data

**Example:**
```
Original Transaction:
- Platinum lot: ₱120,000
- 24-month installment plan
- Down payment: ₱18,000 (15%)
- Schedule created: 24 monthly payments of ₱4,250

Reversal Scenario 1 (ALLOWED):
- No installment payments made yet
- Down payment reversed
- Schedule deleted
- Lot → Available

Reversal Scenario 2 (BLOCKED):
- 3 installment payments already made
- System blocks down payment reversal
- Error: "Must reverse installment payments first"
```

---

#### C. Installment Payment Reversal

**Identification:**
- Not the first payment
- Linked to payment schedule
- Part of Pre-Need installment plan

**Reversal Actions:**
1. Remove installment payment record and documents
2. Revert payment schedule status for affected installment
3. Recalculate remaining balance
4. Update warning level based on current schedule status
5. Keep lot in "Partial" status
6. Maintain ownership information

**Smart Schedule Reversion Logic:**

The system recalculates schedule item status based on current date:

```
For the reversed payment schedule item:
- If due date is in the future → Status: "Pending"
- If due date is today or past → Status: "Overdue"
```

**Example:**
```
Payment Schedule (24 months):
1. Oct 1 - ₱4,250 (Paid, Verified)
2. Nov 1 - ₱4,250 (Paid, Verified)
3. Dec 1 - ₱4,250 (Paid, Verified) ← REVERSING THIS
4. Jan 1 - ₱4,250 (Pending)
5. Feb 1 - ₱4,250 (Pending)
...

Current Date: December 15

After Reversing Dec 1 Payment:
1. Oct 1 - ₱4,250 (Paid, Verified)
2. Nov 1 - ₱4,250 (Paid, Verified)
3. Dec 1 - ₱4,250 (Overdue) ← Status recalculated
4. Jan 1 - ₱4,250 (Pending)
5. Feb 1 - ₱4,250 (Pending)

System Actions:
- Remaining balance increased by ₱4,250
- Warning level recalculated (Dec 1 now overdue by 14 days)
- Lot remains "Partial" status
- Owner unchanged
- Penalties may now apply to Dec 1 payment
```

---

#### D. At-Need Initial Payment Reversal

**Identification:**
- First payment of At-Need purchase
- Amount ≥ 75% of At-Need price (base × 1.5)
- 72-hour deadline applies
- No complex schedule (max 2 payments total)

**Reversal Actions:**
1. Remove initial payment record and documents
2. Delete 72-hour payment deadline
3. Reset lot status to "Available"
4. Clear At-Need designation
5. Reset price to base (remove 1.5x multiplier)
6. Clear ownership information
7. Remove warning Level 7 if applicable

**Example:**
```
Original Transaction:
- Diamond lot base: ₱150,000
- At-Need price: ₱225,000
- Initial payment: ₱168,750 (75%)
- Remaining: ₱56,250
- Due: 72 hours
- Status: Partial

After Reversal:
- Lot Status: Available
- Price reset to base: ₱150,000
- At-Need flag removed
- 72-hour deadline cleared
- Owner: None
- Available for Pre-Need or new At-Need
```

---

#### E. At-Need Remaining Payment Reversal

**Identification:**
- Second payment of At-Need purchase
- Completes the remaining 25% balance
- Made within or after 72-hour deadline
- Changes lot status to "Sold"

**Sequence Validation:**
```
System checks:
1. Is this the last payment in sequence?
2. Is initial At-Need payment still verified?
3. Validation passed → Allow reversal
```

**Reversal Actions:**
1. Remove final payment record and documents
2. Recreate 72-hour deadline from initial payment date
3. Revert lot status from "Sold" to "Partial"
4. Restore remaining balance
5. Recalculate warning level (may trigger Level 7)
6. Maintain At-Need designation
7. Keep ownership information

**Example:**
```
Original Transaction Timeline:
- Monday 10 AM: Initial payment ₱168,750 (75%)
- Wednesday 9 AM: Final payment ₱56,250 (25%)
- Wednesday 9 AM: Status → Sold

Reversal on Wednesday 2 PM:
- Final payment reversed
- Status: Partial
- Balance due: ₱56,250
- New deadline: Thursday 10 AM (72hrs from initial)
- Warning check: Currently 2 hours before Level 7
- Owner: Unchanged
- At-Need designation: Maintained
```

---

### 9.5 Complete Reset Trigger

**Condition:** Balance equals base price after reversal

**Trigger Logic:**
```
After ANY reversal:
IF (Current Balance = Original Base Price) THEN
  Trigger Complete Lot Reset
END IF
```

**Complete Reset Actions:**
1. Change lot status to "Available"
2. Clear all ownership records
3. Delete all payment history
4. Remove all payment documents
5. Cancel all schedules
6. Reset warning level to N/A
7. Clear financial calculations
8. Remove all customer associations

**Example:**
```
Scenario:
- Platinum lot base: ₱120,000
- Only down payment of ₱18,000 was made
- No installments paid yet
- Current balance: ₱102,000

After Down Payment Reversal:
- Balance calculation: ₱102,000 + ₱18,000 = ₱120,000
- System check: ₱120,000 = ₱120,000 (Base Price) ✓
- TRIGGER: Complete Reset
- Final state: Lot Available for new reservation
```

---

### 9.6 Document Cleanup Process

**All reversals perform document cleanup:**

**Documents Affected:**
- Payment receipt PDFs
- Proof of payment images
- Transaction confirmations
- Schedule documents
- Verification documents

**Cleanup Steps:**
1. Identify all documents linked to payment
2. Mark documents for deletion
3. Remove from file storage
4. Clear database references
5. Log deletion for audit trail
6. Maintain audit log (cannot be deleted)

**Audit Trail Preservation:**
```
Even though payment is reversed, system maintains:
- Reversal timestamp
- Admin who performed reversal
- Reason for reversal
- Original payment amount
- Original payment date
- Reversal authorization code

This creates accountability while cleaning up active records.
```

---

### 9.7 Warning Level Recalculation

**After every reversal, system recalculates:**

#### Pre-Need Warning Recalculation

**Process:**
1. Identify all remaining schedule items with "Overdue" status
2. Calculate days overdue for each
3. Find most severe overdue situation
4. Assign corresponding warning level
5. Update lot warning status

**Example:**
```
Before Reversal:
- 3 payments made (all verified)
- Current status: Level 1 (Active)
- Next due: Jan 1

After Reversing Last Payment (Dec 1):
- 2 payments verified
- Dec 1 payment now overdue by 15 days
- System check: 15 days = Level 3 (Overdue)
- Warning level updated: Level 1 → Level 3
- Penalty now applicable
```

#### At-Need Warning Recalculation

**Process:**
1. Check time elapsed since initial payment
2. Compare against 72-hour deadline
3. If deadline passed → Trigger Level 7
4. If deadline not passed → Calculate remaining time

**Example:**
```
Timeline:
- Monday 10 AM: Initial payment ₱168,750
- Deadline: Thursday 10 AM
- Wednesday 11 AM: Final payment ₱56,250 (Completed)
- Wednesday 3 PM: Reversal of final payment

Recalculation:
- Current time: Wednesday 3 PM
- Deadline: Thursday 10 AM
- Time remaining: 19 hours
- Status: Active (no Level 7 yet)
- Warning: "19 hours remaining to complete payment"
```

---

### 9.8 Reversal Processing Workflow

**Step-by-Step System Process:**

**Step 1: Validation**
```
✓ Check payment status = "Verified"
✓ Check not already reversed
✓ Check dependency order (newest first)
✓ Check schedule protection rules
✓ Detect payment type
✓ Verify admin authorization
```

**Step 2: Pre-Reversal Snapshot**
```
- Capture current lot state
- Record current balance
- Save current warning level
- Document current schedule status
- Create audit entry
```

**Step 3: Execute Reversal**
```
- Remove payment record
- Delete associated documents
- Update balance calculations
- Modify/delete schedules as needed
- Update lot status if needed
- Clear ownership if needed
```

**Step 4: Post-Reversal Updates**
```
- Recalculate warning level
- Update schedule statuses
- Recompute penalties if applicable
- Check for complete reset condition
- Update customer notifications
```

**Step 5: Confirmation & Logging**
```
- Generate reversal receipt
- Send notification to customer
- Log complete reversal details
- Update admin dashboard
- Archive reversal documentation
```

---

### 9.9 Error Handling & Validation Messages

**Common Validation Errors:**

| Error | Cause | Resolution |
|-------|-------|------------|
| "Payment not verified" | Payment status is "Pending" | Wait for verification or cancel pending payment |
| "Cannot reverse - newer payments exist" | Dependency order violation | Reverse newer payments first |
| "Down payment protected" | Schedule payments exist | Reverse all installments first |
| "Payment already reversed" | Attempting duplicate reversal | No action needed |
| "Insufficient permissions" | Non-admin attempting reversal | Contact administrator |

**Example Error Scenario:**
```
User attempts to reverse down payment:

System Check:
1. Payment status? Verified ✓
2. Already reversed? No ✓
3. Dependency check? FAIL ✗
   - Found 5 installment payments
   
Error Message:
"Cannot reverse down payment. The following payments must be 
reversed first:
- Nov 1: ₱4,250 (Installment #1)
- Dec 1: ₱4,250 (Installment #2)
- Jan 1: ₱4,250 (Installment #3)
- Feb 1: ₱4,250 (Installment #4)
- Mar 1: ₱4,250 (Installment #5)

Please reverse these payments in order before reversing the 
down payment."
```

---

### 9.10 Best Practices & Guidelines

**For Administrators:**

1. **Always Verify Before Reversing**
   - Confirm with customer
   - Check reason for reversal
   - Document authorization

2. **Follow Dependency Order Strictly**
   - Start with newest payment
   - Work backwards chronologically
   - Verify each step

3. **Communicate with Customer**
   - Explain reversal process
   - Provide timeline
   - Set clear expectations

4. **Document Everything**
   - Reason for reversal
   - Authorization details
   - Customer communication
   - System logs

5. **Monitor Post-Reversal Status**
   - Check warning level updates
   - Verify balance calculations
   - Confirm schedule accuracy

**Security Considerations:**

- Only authorized admins can perform reversals
- All reversals require approval code
- Audit trail cannot be deleted
- Customer notification mandatory
- Financial report updates automatic

---

## 10. Lot Ownership Transfer System

### 10.1 Overview

The lot ownership transfer system enables the legal transfer of lot ownership from one party to another while maintaining complete transaction integrity and administrative oversight. All transfers require administrative approval to ensure proper documentation and compliance.

**Important:** Transfers are permanent actions that change legal ownership. All transfers must be thoroughly verified before approval.

---

### 10.2 Transfer Initiation Rules

The system enforces strict requirements before allowing transfer creation:

| Requirement | Rule | Purpose |
|------------|------|---------|
| **Valid Ownership** | Only current lot owners can initiate transfers | Prevents unauthorized transfers |
| **Staff/Admin Access** | Transfer creation requires staff or admin privileges | Ensures proper authorization |
| **Lot Status Check** | Only Reserved, Partial, or Sold lots can be transferred | Maintains data integrity |

#### Valid Ownership Verification

**System Checks:**
1. Verify requester is listed as current lot owner
2. Check ownership records for accuracy
3. Validate ownership is not in dispute
4. Confirm lot has not been forfeited

**Example - Valid Transfer Request:**
```
Lot: PLAT-001 (Platinum)
Current Owner: Juan dela Cruz
Status: Partial (12 payments made)
Requester: Juan dela Cruz
Staff Member: Maria Santos (Authorized Staff)

✓ Ownership verified
✓ Staff authorized
✓ Lot status eligible
→ Transfer creation allowed
```

**Example - Invalid Transfer Request:**
```
Lot: GOLD-045 (Gold)
Current Owner: Maria Garcia
Status: Sold
Requester: Pedro Reyes (NOT the owner)
Staff Member: Admin User

✗ Ownership verification failed
→ Transfer creation blocked
Error: "Only the current lot owner (Maria Garcia) can initiate this transfer"
```

---

#### Staff/Admin Access Control

**Authorization Levels:**

**Staff Members (Can):**
- Create transfer requests on behalf of owners
- Submit transfer documentation
- View transfer status
- Assist with transfer process

**Staff Members (Cannot):**
- Approve or reject transfers
- Modify approved/rejected transfers
- Override ownership verification

**Administrators (Can):**
- All staff capabilities
- Approve transfer requests
- Reject transfer requests
- View complete transfer history
- Cancel pending transfers
- Document decision rationale

**Example Authorization Flow:**
```
Step 1: Owner visits office
Step 2: Staff verifies owner identity
Step 3: Staff creates transfer request in system
Step 4: System validates ownership
Step 5: Transfer status → Pending
Step 6: Admin reviews and decides
Step 7: System executes decision
```

---

#### Lot Status Eligibility

**Eligible Statuses:**

| Status | Can Transfer? | Considerations |
|--------|---------------|----------------|
| **Available** | ❌ No | No ownership exists |
| **Reserved** | ✅ Yes | Transfer includes reservation rights |
| **Partial** | ✅ Yes | Transfer includes payment obligations |
| **Sold** | ✅ Yes | Full ownership transfer |

**Transfer Implications by Status:**

**Reserved Status Transfer:**
```
Original Situation:
- Lot reserved by Owner A
- 30-minute timer active
- No payments made yet

Transfer Request:
- Owner A → Owner B
- Reservation timer preserved
- Owner B assumes reservation

Post-Approval:
- Reservation continues under Owner B
- Timer not reset
- Owner B must complete initial payment
```

**Partial Status Transfer:**
```
Original Situation:
- Lot: ₱120,000 Platinum
- Paid: ₱45,000 (3 installments)
- Remaining: ₱75,000
- Schedule: 21 payments left
- Warning Level: Level 1 (Active)

Transfer Request:
- Owner A → Owner B
- Payment obligation transfers
- Schedule continues

Post-Approval:
- Owner B assumes all payment obligations
- Payment schedule maintained
- Warning level transferred
- Payment history preserved
- New owner responsible for future payments
```

**Sold Status Transfer:**
```
Original Situation:
- Lot: ₱150,000 Diamond
- Fully paid
- No outstanding balance
- Owner A holds title

Transfer Request:
- Owner A → Owner B
- Clean title transfer

Post-Approval:
- Complete ownership to Owner B
- All rights transferred
- Payment history archived
- New title issued
```

---

### 10.3 Approval Requirements

All transfers must pass through administrative review with strict approval protocols:

#### Admin Authority

**Exclusive Admin Powers:**
- **Approve Transfers:** Grant ownership change
- **Reject Transfers:** Deny ownership change
- **Review Documentation:** Verify all supporting documents
- **Request Additional Info:** Ask for clarification
- **Document Decisions:** Record approval/rejection reasons

**Decision Timeline:**
- No automatic approvals
- Manual review required for every transfer
- Recommended review time: 24-48 hours
- Urgent cases: Same-day review available

---

#### Pending Status Protection

**System Rules:**
```
Only transfers with "Pending" status can be:
- Approved by admin
- Rejected by admin
- Cancelled by admin

Transfers with other statuses are locked:
- Approved → Cannot modify
- Rejected → Cannot reprocess
- Cancelled → Cannot revive
```

**Example - Status Lock:**
```
Scenario 1: Attempting to re-approve
Transfer ID: TRF-2024-001
Status: Approved
Action: Admin tries to approve again

System Response:
✗ "Transfer already approved on Oct 15, 2024
  Cannot modify approved transfers"

Scenario 2: Attempting to process rejected transfer
Transfer ID: TRF-2024-002
Status: Rejected
Action: Admin tries to approve

System Response:
✗ "Transfer was rejected on Oct 16, 2024
  Reason: Incomplete documentation
  Please create new transfer request if needed"
```

---

#### Automatic Ownership Update

**Upon Approval, System Automatically:**

1. **Update Lot Records**
   - Change owner name
   - Update contact information
   - Transfer ownership date
   - Archive previous owner data

2. **Transfer Payment Obligations**
   - Assign schedule to new owner
   - Update payment contact
   - Transfer penalty responsibility
   - Maintain warning level status

3. **Update Financial Records**
   - Link payments to new owner
   - Transfer balance responsibility
   - Update billing information
   - Preserve payment history

4. **Generate Documentation**
   - Create transfer certificate
   - Issue new ownership documents
   - Archive transfer approval
   - Send in-app notifications

**Example - Automatic Update Flow:**
```
Transfer Approved: Oct 20, 2024 10:30 AM

Immediate System Actions:
1. Lot PLAT-015 Owner: Juan Cruz → Maria Santos
2. Contact updated: maria.santos@email.com
3. Schedule reassigned: 18 remaining payments
4. Balance: ₱76,500 now under Maria Santos
5. Warning Level: Level 1 (Active) - maintained
6. Payment history: Preserved and linked
7. Transfer certificate: Generated (TRF-CERT-2024-001)

Notifications Sent (In-App):
- To Juan Cruz: "Transfer approved - ownership transferred"
- To Maria Santos: "Welcome! You are now the owner of Lot PLAT-015"
- To Admin: "Transfer TRF-2024-001 completed successfully"
```

---

#### Reason Recording

**All decisions require documented rationale:**

**Approval Reasons (Examples):**
- "All documentation verified and complete"
- "Ownership transfer validated with legal documents"
- "Family inheritance - death certificate provided"
- "Sale agreement reviewed and approved"

**Rejection Reasons (Examples):**
- "Incomplete documentation - missing ID verification"
- "Outstanding balance disputes unresolved"
- "Current owner identity not verified"
- "Legal documentation insufficient"
- "Pending legal issues on the lot"

**System Requirements:**
- Minimum 20 characters for reason
- Cannot use generic reasons repeatedly
- Must be specific to the case
- Cannot approve/reject without reason
- Reason permanently recorded

**Example Documentation:**
```
Transfer Request: TRF-2024-045
From: Pedro Reyes → Ana Reyes
Lot: GOLD-032
Status: Partial (₱35,000 paid, ₱40,000 remaining)

Admin Review:
Date: Oct 18, 2024
Reviewer: Admin Maria Cruz
Decision: APPROVED

Reason:
"Transfer approved. Family transfer verified with:
- Marriage certificate provided
- Both parties present at office
- Photo IDs verified
- Payment responsibility acknowledged by Ana Reyes
- All required signatures obtained
- No outstanding disputes"

Documentation Attached:
- Marriage certificate (scan)
- Pedro Reyes ID (scan)
- Ana Reyes ID (scan)
- Transfer agreement (signed)
```

---

### 10.4 Transfer Status Definitions

The system uses four distinct statuses to track transfer lifecycle:

| Status | Meaning | Possible Next Actions | Can Modify? |
|--------|---------|----------------------|-------------|
| **Pending** | Awaiting admin review | Approve, Reject, Cancel | Yes (Admin only) |
| **Approved** | Transfer completed successfully | None (Final) | No |
| **Rejected** | Transfer denied by admin | Create new request | No |
| **Cancelled** | Transfer cancelled before processing | Create new request | No |

---

#### Pending Status

**Definition:** Transfer request submitted and awaiting administrative review

**Characteristics:**
- Initial status for all new transfers
- Open for admin decision
- Can be approved or rejected
- Can be cancelled by admin
- No ownership changes yet

**Available Actions:**
- Admin: Approve, Reject, Cancel
- Staff: View, add documentation
- Owner: View status

**Example:**
```
Transfer: TRF-2024-078
Created: Oct 18, 2024 9:15 AM
From: Jose Santos
To: Linda Santos
Lot: PLAT-023 (Partial - ₱88,000 paid)
Status: PENDING
Days Pending: 2 days

Admin Dashboard Shows:
🟡 Pending Review
📄 Documents: 3 attached
📝 Staff Notes: "Family transfer - daughter taking over payments"
⏰ Pending Since: 2 days ago
👤 Assigned Reviewer: [Not assigned]

Actions Available:
[✅ Approve] [❌ Reject] [⚫ Cancel]
```

---

#### Approved Status

**Definition:** Transfer has been approved and ownership has been changed

**Characteristics:**
- Final status (cannot be changed)
- Ownership already transferred
- System updates completed
- New owner now responsible
- Original transfer request locked

**Available Actions:**
- All users: View only
- System: No modifications allowed

**Approval Effects:**
1. Lot ownership record updated
2. Payment schedules reassigned
3. Balance responsibility transferred
4. Warning level maintained
5. Transfer certificate generated
6. In-app notifications sent
7. Audit log entry created

**Example:**
```
Transfer: TRF-2024-078
Created: Oct 18, 2024 9:15 AM
Approved: Oct 20, 2024 11:45 AM
From: Jose Santos
To: Linda Santos
Lot: PLAT-023
Status: APPROVED ✅

Approval Details:
Approved By: Admin Maria Cruz
Approval Date: Oct 20, 2024 11:45 AM
Reason: "All documentation verified. Family transfer 
         approved with payment acknowledgment signed."

Post-Approval State:
Current Owner: Linda Santos
Balance: ₱32,000 remaining
Schedule: 8 payments left
Warning Level: Level 1 (Active)
Next Payment Due: Nov 1, 2024

🔒 Transfer completed and locked
```

---

#### Rejected Status

**Definition:** Transfer request has been denied by administrator

**Characteristics:**
- Final status (cannot be changed)
- No ownership change occurred
- Original owner retained
- Documented reason required
- Can create new request if issues resolved

**Available Actions:**
- All users: View rejection reason
- Admin: Cannot modify
- Staff: Can create new transfer if appropriate

**Rejection Reasons:**
- Documentation issues
- Verification failures
- Unresolved disputes
- Legal complications
- Missing information

**Example:**
```
Transfer: TRF-2024-082
Created: Oct 17, 2024 2:30 PM
Rejected: Oct 19, 2024 10:20 AM
From: Carlos Mendoza
To: Unknown Recipient
Lot: DIAM-008
Status: REJECTED ❌

Rejection Details:
Rejected By: Admin Roberto Cruz
Rejection Date: Oct 19, 2024 10:20 AM
Reason: "Transfer rejected due to incomplete documentation.
         Missing:
         - Recipient's valid ID
         - Notarized transfer agreement
         - Payment acknowledgment form
         
         Please resubmit with complete documents."

Current State:
Owner: Carlos Mendoza (unchanged)
Lot Status: Partial
Balance: ₱112,500 remaining

Action Required:
Staff can create new transfer request once 
documents are complete.
```

---

#### Cancelled Status

**Definition:** Transfer request withdrawn before processing

**Characteristics:**
- Transfer stopped before approval/rejection
- No ownership change
- Original owner retained
- Less formal than rejection
- Can be cancelled by admin

**Common Cancellation Reasons:**
- Owner changed mind
- Duplicate request
- Error in transfer details
- Alternative resolution found
- Request withdrawn by owner

**Example:**
```
Transfer: TRF-2024-090
Created: Oct 18, 2024 4:00 PM
Cancelled: Oct 18, 2024 5:15 PM
From: Ana Garcia
To: Pedro Garcia
Lot: GOLD-045
Status: CANCELLED ⚫

Cancellation Details:
Cancelled By: Admin Juan Reyes
Cancellation Date: Oct 18, 2024 5:15 PM
Reason: "Owner requested cancellation - decided to 
         handle transfer through legal process instead.
         May resubmit after legal documents finalized."

Current State:
Owner: Ana Garcia (unchanged)
Lot Status: Sold
No further action on this transfer request.

Note: New transfer can be created if needed.
```

---

### 10.5 Transfer Process Workflow

**Complete step-by-step transfer process:**

#### Step 1: Transfer Initiation

**Prerequisites:**
```
✓ Current owner verified
✓ Staff/Admin authorized
✓ Lot status eligible (Reserved/Partial/Sold)
✓ Required documents prepared
```

**Actions:**
1. Staff logs into system
2. Searches for lot by number or owner
3. Clicks "Initiate Transfer"
4. System validates eligibility
5. Staff enters transfer details:
   - New owner information
   - Contact details
   - Reason for transfer
   - Supporting documents
6. System creates transfer record
7. Status set to "Pending"
8. In-app notification sent to admin

**Example Entry:**
```
Transfer Request Creation:
Lot: PLAT-025
Current Owner: Maria Santos
New Owner: Carlos Santos
Relationship: Spouse
Transfer Reason: Family arrangement
Documents Attached: 2 files
Created By: Staff Member - Ana Cruz
Status: Pending

In-App Notification Sent To:
- All Administrators
- Original Owner (Maria Santos)
```

---

#### Step 2: Admin Review

**Admin Dashboard View:**
```
New Transfer Request
TRF-2024-095
🟡 Pending Review - 0 days old

Lot Details:
Number: PLAT-025
Classification: Platinum (₱120,000)
Status: Partial
Balance: ₱54,000 remaining

Transfer Details:
From: Maria Santos
To: Carlos Santos
Reason: Family arrangement
Documents: 2 attached

Actions:
[View Documents] [Approve] [Reject] [Cancel]
```

**Review Checklist:**
- [ ] Verify current owner identity
- [ ] Review all attached documents
- [ ] Check lot payment status
- [ ] Verify new owner information
- [ ] Review any outstanding issues
- [ ] Confirm transfer reason legitimacy
- [ ] Check for legal complications

---

#### Step 3: Admin Decision

**Option A: Approval**

**Admin Actions:**
1. Reviews all documentation
2. Verifies compliance
3. Clicks "Approve"
4. Enters approval reason (required)
5. Confirms decision

**System Actions:**
1. Changes transfer status to "Approved"
2. Updates lot owner to Carlos Santos
3. Transfers payment schedule
4. Updates contact information
5. Generates transfer certificate
6. Records approval in audit log
7. Sends in-app notifications:
   - To Maria Santos: "Transfer approved"
   - To Carlos Santos: "You are now the owner"
   - To Staff Member: "Transfer completed"
   - To Admin: "Transfer successful"

**Option B: Rejection**

**Admin Actions:**
1. Identifies issues
2. Clicks "Reject"
3. Enters detailed rejection reason (required)
4. Confirms decision

**System Actions:**
1. Changes transfer status to "Rejected"
2. Records rejection reason
3. Preserves original ownership
4. Sends in-app notifications:
   - To Maria Santos: "Transfer rejected - [reason]"
   - To Staff Member: "Transfer rejected - [reason]"
   - To Admin: "Transfer rejected successfully"

**Option C: Cancellation**

**Admin Actions:**
1. Determines cancellation needed
2. Clicks "Cancel"
3. Enters cancellation reason
4. Confirms decision

**System Actions:**
1. Changes transfer status to "Cancelled"
2. Records cancellation reason
3. Preserves original ownership
4. Sends in-app notifications
5. Archives transfer request

---

### 10.6 Notification System (In-App Only)

**All transfer notifications are delivered through in-app notification center only. No email or SMS notifications.**

#### Notification Types

**Transfer Created:**
```
📬 New Transfer Request
Transfer ID: TRF-2024-095
Lot: PLAT-025
From: Maria Santos → Carlos Santos
Status: Pending Admin Review
Created: Oct 20, 2024 10:30 AM

Action Required: Admin review needed
[View Details]
```

**Transfer Approved:**
```
✅ Transfer Approved
Transfer ID: TRF-2024-095
Lot: PLAT-025 is now owned by Carlos Santos
Previous Owner: Maria Santos
Approved: Oct 22, 2024 2:15 PM
Approved By: Admin Juan Cruz

Your new ownership is now active.
[View Transfer Certificate]
```

**Transfer Rejected:**
```
❌ Transfer Rejected
Transfer ID: TRF-2024-096
Lot: GOLD-033
Reason: Incomplete documentation - missing recipient ID

Action Required: Please provide missing documents
and resubmit transfer request.
[View Rejection Details]
```

**Transfer Cancelled:**
```
⚫ Transfer Cancelled
Transfer ID: TRF-2024-097
Lot: DIAM-015
Reason: Owner request - handling through legal process

No further action on this request.
[View Details]
```

#### Notification Recipients

| Event | Current Owner | New Owner | Staff Member | Admin |
|-------|---------------|-----------|--------------|-------|
| Transfer Created | ✅ Notified | ❌ No | ✅ Notified | ✅ Notified |
| Transfer Approved | ✅ Notified | ✅ Notified | ✅ Notified | ✅ Notified |
| Transfer Rejected | ✅ Notified | ❌ No | ✅ Notified | ✅ Notified |
| Transfer Cancelled | ✅ Notified | ❌ No | ✅ Notified | ✅ Notified |

---

### 10.7 Transfer Impact on Payment Obligations

#### Payment Schedule Transfer

**When transfer is approved:**

**Pre-Need Installments:**
```
Before Transfer:
Owner: Juan Santos
Schedule: 15 remaining payments
Monthly: ₱4,250
Next Due: Nov 1, 2024
Warning Level: Level 1

After Transfer:
Owner: Maria Santos (NEW)
Schedule: 15 remaining payments (unchanged)
Monthly: ₱4,250 (unchanged)
Next Due: Nov 1, 2024 (unchanged)
Warning Level: Level 1 (maintained)

New owner assumes all payment obligations immediately.
```

**At-Need Remaining Payment:**
```
Before Transfer:
Owner: Pedro Cruz
At-Need: ₱225,000 total
Paid: ₱168,750 (75%)
Remaining: ₱56,250
Due: 72 hours from initial (Oct 25, 10 AM)

After Transfer (Oct 23, 2 PM):
Owner: Ana Cruz (NEW)
Remaining: ₱56,250 (unchanged)
Due: Oct 25, 10 AM (unchanged - 44 hours left)
New owner must complete payment within remaining time.
```

---

#### Warning Level Preservation

**System maintains warning levels through transfers:**

**Example:**
```
Transfer During Overdue Period:

Original Situation:
Owner: Carlos Mendoza
Warning Level: Level 4 (First Warning)
Days Overdue: 45 days
Penalty: ₱4,500
Balance: ₱52,000

Transfer Approved:
New Owner: Linda Mendoza
Warning Level: Level 4 (maintained)
Days Overdue: 45 days (continues)
Penalty: ₱4,500 (transferred)
Balance: ₱52,000 (transferred)

Important: New owner inherits the overdue status
and must address penalties immediately.
```

---

### 10.8 Best Practices & Guidelines

**For Staff Members:**

1. **Verify Identity Thoroughly**
   - Check government-issued IDs
   - Confirm ownership in system
   - Match signatures
   - Document verification process

2. **Complete Documentation**
   - Collect all required documents
   - Scan clearly
   - Attach to transfer request
   - Note any missing items

3. **Clear Communication**
   - Explain transfer process to owners
   - Set realistic timelines
   - Provide status updates
   - Answer questions clearly

4. **Accurate Data Entry**
   - Double-check new owner details
   - Verify contact information
   - Review payment status
   - Confirm lot details

**For Administrators:**

1. **Thorough Review**
   - Review all documentation
   - Verify authenticity
   - Check for inconsistencies
   - Consult legal if needed

2. **Timely Processing**
   - Review within 24-48 hours
   - Don't let requests accumulate
   - Prioritize urgent cases
   - Communicate delays

3. **Detailed Reasoning**
   - Provide specific approval reasons
   - Explain rejections clearly
   - Document decision factors
   - Maintain consistency

4. **Monitor Post-Transfer**
   - Verify ownership update
   - Check payment reassignment
   - Confirm notification delivery
   - Address any issues quickly

**Security Considerations:**

- All transfers require multi-factor authentication
- Document all access and changes
- Maintain complete audit trail
- Protect sensitive owner information
- Secure transfer documentation
- Monitor for fraudulent requests

---

## 11. Component Workflows

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

| Setting | Default Value | Description |
|---------|---------------|-------------|
| Reservation Timeout | 30 minutes | Auto-release reserved lots |
| Grace Period | 7 days | No-penalty period after due date |
| Penalty Rate | 2% monthly | Monthly penalty percentage |

### 10.2 Pre-Need Settings

| Setting | Default Value | Description |
|---------|---------------|-------------|
| Minimum Down Payment | 15% | Minimum initial payment |
| Spot Cash 7-day Discount | 10% | Full payment within 7 days |
| Spot Cash 15-day Discount | 7% | Full payment within 15 days |
| Spot Cash 30-day Discount | 5% | Full payment within 30 days |
| Forfeiture Threshold | 90 days | Days overdue before forfeiture eligibility |

### 10.3 At-Need Settings

| Setting | Default Value | Description |
|---------|---------------|-------------|
| Price Multiplier | 1.5x | Urgency pricing factor |
| Minimum Down Payment | 75% | Minimum initial payment |
| Payment Deadline | 72 hours | Time to complete payment |
| Manual Review Trigger | 72 hours | When Level 7 activates |

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

| Current Status | Cash Full Payment | Cash Partial | Non-Cash Full | Non-Cash Partial |
|---------------|-------------------|--------------|---------------|------------------|
| Available | → Sold (Verified) | → Partial (Verified) | → Partial (Pending) | → Partial (Pending) |
| Reserved | → Sold (Verified) | → Partial (Verified) | → Partial (Pending) | → Partial (Pending) |
| Partial | → Sold (Verified) | → Partial (Verified) | → Partial (Pending) | → Partial (Pending) |

### Payment Method Verification

| Method | Verification | Admin Approval Required |
|--------|--------------|------------------------|
| Cash | Automatic | No |
| GCash | Manual | Yes |
| Bank Transfer | Manual | Yes |
| Check | Manual | Yes |
| Other Non-Cash | Manual | Yes |

---

## Document Control

**Version:** 3.0
**Last Updated:**  10/18/2025  
**Maintained By:** Erick
