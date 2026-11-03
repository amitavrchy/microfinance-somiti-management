# Microfinance / Somiti Management — Project Spec

> Stack: **Next.js (field PWA) + NestJS + Prisma + PostgreSQL + Redis/BullMQ**  
> Bangladesh somiti/MFI: members, groups, loans with **kisti** schedules, savings, flat & declining interest, dues/overdue, field officer collection app, audit.

---

## 1. Overview & Goals

### Product
Digitize cooperative/microfinance workflows: member KYC, samity/groups, savings deposits/withdrawals, loan origination, installment (kisti) generation, field collection with offline sync, and branch audit trails.

### Engineering goals
- Interest engines: **flat** and **declining balance** with correct schedules.
- Idempotent collection posting; reverse with contra entries (no silent edit).
- Overdue ageing & penalty rules.
- Field officer route list + offline collections.
- Immutable ledger for savings & loan accounts (double-entry style).

### Non-goals
- Full CBS core banking / mobile financial institution license stack.
- Credit bureau integrations (adapter stub).

---

## 2. User Roles & Permissions (RBAC)

| Role | Scope |
|------|-------|
| `HQ_ADMIN` | Products, branches, policies |
| `BRANCH_MANAGER` | Approvals above limit, officers, reports |
| `LOAN_OFFICER` / `FIELD_OFFICER` | Collections, applications intake |
| `CASHIER` | Branch cash collections |
| `AUDITOR` | Read + freeze flags |
| `MEMBER` (optional portal) | Self balance, schedule |

Maker-checker: loan disburse & large withdrawal.

---

## 3. Feature Modules

1. Branches & field officer territories  
2. Members & KYC / nominees  
3. Groups / samity meetings  
4. Savings products & accounts  
5. Loan products (flat/declining), applications, disbursement  
6. Kisti schedules & collections  
7. Penalties, restructuring, write-off  
8. Field collection PWA + offline sync  
9. Cashbook / vault  
10. Audit & regulatory reports (PAR, collection efficiency)

---

## 4. Advanced Business Logic

### 4.1 Flat interest loan
```
interestTotal = principal * annualRate * termYears
totalPayable = principal + interestTotal
installment = totalPayable / numKisti   // last kisti adjusts remainder paisa
```
Schedule: equal kisti; components split principal/interest by product rule (equal principal or interest-first).

### 4.2 Declining balance
```
For each period:
  interest = outstandingPrincipal * periodicRate
  principalComponent = installment - interest  // or fixed principal method
  outstanding -= principalComponent
```
Support reducing-balance with fixed installment (EMI-like) using standard amortization formula.

### 4.3 Collection posting
```
BEGIN;
 lock LoanAccount;
 allocate payment: penalty → interest → principal (configurable waterfall)
 create Collection + LoanLedgerEntry entries
 update outstanding, nextDue, daysPastDue
COMMIT;
```
Idempotency: `clientCollectionId` unique per officer device.

### 4.4 Savings
- Min balance; withdrawal limits; interest posting job (monthly) on minimum/daily balance product config.
- Guarantor/lien: block savings withdrawal if linked as loan security.

### 4.5 Overdue & PAR
- DPD = today - oldest unpaid kisti due date.
- PAR30/60/90 portfolio aggregates.
- Penalty = overduePrincipalOrInstallment * penaltyRate * days (cap).

### 4.6 Edge cases
- Early settlement rebate rules.  
- Partial kisti payment leaves remainder due.  
- Holiday calendar shifts due dates (BD weekends/govt holidays).  
- Group liability: optional joint liability flag when member defaults.  
- Disburse only after approval + cash vault check.  
- Reverse collection same day only; after day-end requires manager + audit reason.

---

## 5. State Machines

**Member**: `APPLIED → ACTIVE | REJECTED | CLOSED | BLACKLISTED`  
**LoanApplication**: `DRAFT → SUBMITTED → UNDER_REVIEW → APPROVED → DISBURSED | REJECTED`  
**LoanAccount**: `ACTIVE → CLOSED | WRITTEN_OFF | RESTRUCTURED`  
**Kisti**: `UPCOMING → DUE → PARTIAL → PAID | OVERDUE`  
**CollectionBatch**: `OPEN → SYNCED → DAY_CLOSED`  
**SavingsWithdrawal**: `REQUESTED → APPROVED → PAID | REJECTED`

---

## 6. Prisma Schema Draft

```prisma
model Branch { id String @id @default(cuid()) code String @unique name String }
model User {
  id String @id @default(cuid())
  branchId String?
  role String
  fullName String
  phone String? @unique
  passwordHash String?
}
model Member {
  id String @id @default(cuid())
  branchId String
  memberCode String
  name String
  nidHash String?
  phone String?
  status String
  kyc Json?
  @@unique([branchId, memberCode])
}
model GroupSamity {
  id String @id @default(cuid())
  branchId String
  name String
  meetingDay Int?
  officerId String?
}
model GroupMember {
  groupId String
  memberId String
  @@id([groupId, memberId])
}
model SavingsProduct {
  id String @id @default(cuid())
  name String
  interestRateBps Int
  interestMethod String // MIN_BALANCE DAILY
  minBalanceMinor Int @default(0)
}
model SavingsAccount {
  id String @id @default(cuid())
  memberId String
  productId String
  balanceMinor Int @default(0)
  status String
}
model SavingsLedgerEntry {
  id String @id @default(cuid())
  accountId String
  type String
  debitMinor Int @default(0)
  creditMinor Int @default(0)
  balanceAfterMinor Int
  refType String?
  refId String?
  createdAt DateTime @default(now())
}
model LoanProduct {
  id String @id @default(cuid())
  name String
  interestMethod String // FLAT DECLINING
  annualRateBps Int
  termMonths Int
  kistiCount Int
  penaltyRateBps Int @default(0)
  paymentWaterfall String @default("PENALTY_INTEREST_PRINCIPAL")
}
model LoanApplication {
  id String @id @default(cuid())
  memberId String
  productId String
  principalMinor Int
  status String
  guarantors Json?
}
model LoanAccount {
  id String @id @default(cuid())
  applicationId String @unique
  memberId String
  productId String
  principalMinor Int
  outstandingPrincipalMinor Int
  outstandingInterestMinor Int
  outstandingPenaltyMinor Int @default(0)
  status String
  disbursedAt DateTime?
  @@index([memberId, status])
}
model KistiInstallment {
  id String @id @default(cuid())
  loanId String
  seq Int
  dueDate DateTime @db.Date
  principalDueMinor Int
  interestDueMinor Int
  penaltyDueMinor Int @default(0)
  principalPaidMinor Int @default(0)
  interestPaidMinor Int @default(0)
  penaltyPaidMinor Int @default(0)
  status String
  @@unique([loanId, seq])
  @@index([dueDate, status])
}
model Collection {
  id String @id @default(cuid())
  loanId String?
  savingsAccountId String?
  officerId String
  amountMinor Int
  collectedAt DateTime
  clientCollectionId String @unique
  method String // CASH BKASH NAGAD
  reversedAt DateTime?
}
model CollectionAllocation {
  id String @id @default(cuid())
  collectionId String
  kistiId String?
  component String // PENALTY INTEREST PRINCIPAL SAVINGS
  amountMinor Int
}
model Holiday {
  id String @id @default(cuid())
  date DateTime @db.Date
  name String
  @@unique([date])
}
model AuditLog {
  id String @id @default(cuid())
  actorId String?
  action String
  entity String
  entityId String?
  meta Json?
  createdAt DateTime @default(now())
}
model OutboxEvent {
  id String @id @default(cuid())
  type String
  payload Json
  processedAt DateTime?
  createdAt DateTime @default(now())
}
```

---

## 7. REST API

- Members/groups CRUD  
- Savings deposit/withdraw  
- Loan apply/approve/disburse  
- `GET /loans/:id/schedule`  
- `POST /collections` (idempotent)  
- `POST /collections/:id/reverse`  
- Field: `GET /field/my-dues`, `POST /sync/collections`  
- Reports: PAR, collection efficiency, officer productivity  
- Day-end close branch  

---

## 8. Real-time

Optional branch cashier feed of field sync collections. Not critical offline.

---

## 9. Jobs

- Mark kisti OVERDUE nightly; accrue penalty  
- Savings interest posting  
- PAR snapshot  
- Offer expiry N/A; loan approval SLA reminders  
- Holiday-aware due date shift on schedule generation  

---

## 10. Security

- Field officer sees only assigned groups/members.  
- Maker-checker for disburse.  
- NID stored hashed/encrypted.  
- Reverse collection audited.

---

## 11. Caching

- Officer daily dues list cached; invalidate on collection.  
- Product config cache.

---

## 12. Testing

- Flat vs declining schedule golden fixtures (paisa remainder on last kisti).  
- Waterfall allocation order.  
- Parallel collections idempotent + outstanding never negative.  
- Lien blocks savings withdrawal.  
- Day-end prevents officer backdated collection without manager.

---

## 13. Folder Structure

```
microfinance-somiti-management/
  Project_spec.md
  frontend/  # branch + field PWA
  backend/
```

---

## 14. Env vars

`DATABASE_URL`, `REDIS_URL`, `JWT_*`, `APP_URL`, `CORS_ORIGINS`, `S3_*`, `SMTP_URL`, `SMS_*`, `LOAN_APPROVAL_LIMIT_MINOR`, `COLLECTION_REVERSE_SAME_DAY_ONLY`, `PAR_BUCKETS`, `OFFLINE_COLLECTION_MAX_HOURS`

---

## 15. Milestones

1. Branches, members, groups, KYC  
2. Savings ledger  
3. Loan products + schedule generators  
4. Disburse + collections waterfall  
5. Overdue/penalties/PAR  
6. Field offline sync  
7. Audit reports + day-end  

---

## 16. Acceptance Criteria

- [ ] Flat and declining schedules match fixture sheets to the paisa.
- [ ] Duplicate `clientCollectionId` does not double-apply.
- [ ] Waterfall pays penalty before interest before principal.
- [ ] PAR30 count correct after freezing clock in tests.
- [ ] Savings linked as security cannot withdraw below lien.
- [ ] Field officer cannot collect for other branch members.
- [ ] Seed: 1 branch, 2 groups, 50 members, sample active loans.

---

## 17. Agent Notes

- Pure functions for schedule generation in `InterestEngine`.  
- Never update ledger balances without inserting ledger rows.  
- Bangla labels for kisti receipts; print-friendly collection sheet.
