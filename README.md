# Backend Migration: Frontend Filters & Calculations Audit

> **Purpose:** Documents every filter and calculation currently done on the frontend (client side) that must be moved to the backend with proper pagination support when scaling.

---

## Table of Contents

1. [Billing List API](#1-billing-list-api)
2. [Cancelled Billing List API](#2-cancelled-billing-list-api)
3. [Expense (Petty-Cash-Expense) List API](#3-expense-petty-cash-expense-list-api)
4. [Petty Cash List API](#4-petty-cash-list-api)
5. [Income / Booking Transaction List API](#5-income--booking-transaction-list-api)
6. [User Ledger / Transaction History API](#6-user-ledger--transaction-history-api)
7. [Balance Sheet API](#7-balance-sheet-api)
8. [Summary Table](#8-summary-table)

---

## 1. Billing List API

**Endpoint:** `GET /billing`  
**Provider File:** `lib/modules/billing/provider/billing_provider.dart`  
**Screen File:** `lib/modules/billing/screen/billing_screen.dart`

### 1.1 Filters (Done on Frontend)

| # | Filter Name | Logic | Field(s) Used |
|---|-------------|-------|---------------|
| 1 | **Date Range Filter** | Filters bills where `eventDate >= startDate` AND `eventDate <= endDate` | `bill.eventDate` |
| 2 | **Search Filter** | Case-insensitive `contains` on party-plot name, created-by username, and lead name | `bill.partyPlot.name`, `bill.createdBy.username`, `bill.bookingData.lead.name` |
| 3 | **Payment Status Filter** | Computed status (`PENDING`, `CLEAR`, `EXTRA`, `CANCELLED`) and multi-select filter on it | Derived from `totalAmount`, `gstAmount`, `paidAmount`, `billing_status` |
| 4 | **Sort by Event Date** | After fetching, list is sorted ascending by `eventDate` | `bill.eventDate` |

### 1.2 Calculations (Done on Frontend)

| # | Calculation | Formula | Where Used |
|---|-------------|---------|-----------|
| 1 | **Computed Bill Status** | `status = 'CANCELLED'` if `billing_status == 'CANCELLED'`; else `outstanding = (totalAmount + gstAmount) - paidAmount`; if `outstanding == 0` → `CLEAR`; if `> 0` → `PENDING`; else → `EXTRA` | `BillingProvider._computeStatus()` (line 54–59), `billing_screen.dart` (line 789–794) |
| 2 | **Outstanding Amount Display** | `outstanding = billing.totalAmount - billing.paidAmount` (screen uses `totalAmount` only, provider uses `totalAmount + gstAmount`) | `billing_screen.dart` (line 680, 792) — **Note: inconsistency exists between provider and screen formulas** |
| 3 | **GST Back-calculation** | If `gst_amount == 0` and `bank_amount > 0`, then `gstVal = bankAmount - (bankAmount / 1.18)` | `BillingModel.fromJson()` (line 52–55), `copyWith()` (line 111–112) |
| 4 | **Excel Total Amount per Row** | `total = packageAmount + extraDecorAmount + extraAgencyAmount + gstAmount` | `BillingProvider.exportBillingSheetSyncfusion()` (line 398–402) |
| 5 | **Status Color Logic** | Outstanding == 0 → green; > 0 → red; < 0 → orange | `billing_screen.dart` `_getOutstandingColor()` (line 849–881) |

### 1.3 Required Backend Query Parameters

```
GET /billing
  ?startDate=YYYY-MM-DD
  &endDate=YYYY-MM-DD
  &search=<party_plot_name|username|lead_name>
  &status=PENDING|CLEAR|EXTRA|CANCELLED   (multi-value)
  &sortBy=event_date
  &sortOrder=asc|desc
  &page=1
  &limit=20
```

### 1.4 Required Backend Computed Fields in Response

- `computed_status`: `PENDING | CLEAR | EXTRA | CANCELLED`
- `outstanding_amount`: `(package_amount + extra_decor_amount + extra_agency_amount + gst_amount) - paid_amount`
- `gst_amount`: Always store actual GST; remove frontend back-calculation

---

## 2. Cancelled Billing List API

**Endpoint:** `GET /billing/cancelled`  
**Provider File:** `lib/modules/cancelled_billing/provider/cancelled_billing_provider.dart`  
**Screen File:** `lib/modules/cancelled_billing/screen/cancelled_billing_screen.dart`

### 2.1 Filters (Done on Frontend)

| # | Filter Name | Logic | Field(s) Used |
|---|-------------|-------|---------------|
| 1 | **Date Range Filter** | Filters by `eventDate >= startDate` AND `eventDate <= endDate` | `bill.eventDate` |
| 2 | **Search Filter** | Case-insensitive `contains` on party-plot name, lead name, or created-by username | `bill.partyPlot.name`, `bill.bookingData.lead.name`, `bill.createdBy.username` |
| 3 | **Sort by Event Date** | After fetching, sorted descending (newest first) by `eventDate` | `bill.eventDate` |

### 2.2 Calculations (Done on Frontend — Screen)

| # | Calculation | Formula | Where Used |
|---|-------------|---------|-----------|
| 1 | **Total Refundable Amount** | `totalRefundable += bill.paidAmount` (summed for all visible cancelled bills) | `cancelled_billing_screen.dart` (line 233) |
| 2 | **Total Waived Amount** | `totalWaived += bill.outstandingAmount` (summed for all visible cancelled bills) | `cancelled_billing_screen.dart` (line 234) |

### 2.3 Required Backend Query Parameters

```
GET /billing/cancelled
  ?startDate=YYYY-MM-DD
  &endDate=YYYY-MM-DD
  &search=<party_plot_name|lead_name|username>
  &sortBy=event_date
  &sortOrder=desc
  &page=1
  &limit=20
```

### 2.4 Required Backend Computed Fields in Response

- Pagination metadata: `total`, `page`, `limit`, `totalPages`
- Aggregate fields (optional, for summary row): `total_refundable`, `total_waived`

---

## 3. Expense (Petty-Cash-Expense) List API

**Endpoint:** `GET /petty-cash-expense/`  
**Provider File:** `lib/modules/expanse/provider/expense_provider.dart`

### 3.1 Filters (Done on Frontend)

| # | Filter Name | Logic | Field(s) Used |
|---|-------------|-------|---------------|
| 1 | **Role-based Filter (FINANCE_MANAGER)** | If token role is `FINANCE_MANAGER`, keep only records where `expense.managerId == tokenData['_id']` | `expense.managerId`, JWT `_id` |
| 2 | **Role-based Filter (EXPENSE role)** | If token role is `EXPENSE`, keep only records where `expense.partyPlotId` is in `allowedPartyPlotIds` list | `expense.partyPlotId`, `allowedPartyPlotIds` |
| 3 | **Search by Person Name** | Case-insensitive `contains` on `person_name` | `expense.person_name` |
| 4 | **Search by Description** | Case-insensitive `contains` on `expenseDescription` | `expense.expenseDescription` |
| 5 | **Manager Name Filter** | Case-insensitive `contains` on `managerName` | `expense.managerName` |
| 6 | **Party Plot Filter** | Case-insensitive `contains` on `partyPlotName` | `expense.partyPlotName` |
| 7 | **Head Category Filter** | Case-insensitive `contains` on `headCategoryName` | `expense.headCategoryName` |
| 8 | **Sub-Head Category Filter** | Case-insensitive `contains` on `subHeadCategoryName` (only when head is also set) | `expense.subHeadCategoryName` |
| 9 | **Date Range Filter** | Filters by `createdAt >= startDate` AND `createdAt <= endDate` (endDate set to 23:59:59) | `expense.createdAt` |

### 3.2 Calculations (Done on Frontend)

> No complex amount calculations in this module. Amounts are stored as-is from the API. The Excel export columns (balance before/after) are provided by the backend's download endpoint.

### 3.3 Required Backend Query Parameters

```
GET /petty-cash-expense/
  ?managerId=<id>              (replaces FINANCE_MANAGER role filter)
  &partyPlotIds=id1,id2        (replaces EXPENSE role filter)
  &personName=<search>
  &description=<search>
  &managerName=<search>
  &partyPlot=<search>
  &head=<head_category_name>
  &subHead=<sub_head_category_name>
  &startDate=YYYY-MM-DD
  &endDate=YYYY-MM-DD
  &page=1
  &limit=20
```

---

## 4. Petty Cash List API

**Endpoint:** `GET /petty-cash/get`  
**Provider File:** `lib/modules/peti_cash/provider/peti_cash_provider.dart`

### 4.1 Filters (Done on Frontend)

| # | Filter Name | Logic | Field(s) Used |
|---|-------------|-------|---------------|
| 1 | **Role-based Filter (FINANCE_MANAGER)** | Keep only records where `petiCash.managerId == tokenData['_id']` | `petiCash.managerId`, JWT `_id` |
| 2 | **Role-based Filter (ACCOUNTANT)** | Keep only records where `petiCash.managerId == tokenData['_id']` | `petiCash.managerId`, JWT `_id` |
| 3 | **Role-based Filter (ADMIN)** | Exclude records where `petiCash.managerId == tokenData['_id']` (admin sees everyone else's) | `petiCash.managerId`, JWT `_id` |
| 4 | **Search Filter** | Case-insensitive `contains` on `managerName` OR `allocatedAmount` (as string) | `petiCash.managerName`, `petiCash.allocatedAmount` |
| 5 | **Date Range Filter** | Filters by `petiCash.date >= startDate` AND `petiCash.date <= endDate` | `petiCash.date` |

### 4.2 Calculations (Done on Frontend)

> No amount calculations. The amounts are direct values from the API.

### 4.3 Required Backend Query Parameters

```
GET /petty-cash/get
  ?search=<manager_name|amount>
  &startDate=YYYY-MM-DD
  &endDate=YYYY-MM-DD
  &page=1
  &limit=20
```

> **Note:** Role-based filtering should be handled server-side using the authenticated user's JWT. The backend should automatically scope results based on role (`ADMIN` sees all, `FINANCE_MANAGER` / `ACCOUNTANT` see only their own).

---

## 5. Income / Booking Transaction List API

**Endpoint (Booking List):** `GET /booking/transaction` (with `startDate`, `endDate` params)  
**Endpoint (Transaction by User):** `GET /user-ledger/transactions/:userId`  
**Provider File:** `lib/modules/income/provider/income_provider.dart`

### 5.1 Filters (Done on Frontend)

| # | Filter Name | Logic | Field(s) Used |
|---|-------------|-------|---------------|
| 1 | **Party Plot Filter** | Exact match on `transaction.party_plot_name` | `transaction.party_plot_name` |
| 2 | **Function Date Filter** | Exact match after formatting: `DateFormatter.format(transaction.booking_date) == functionDate` | `transaction.booking_date` |
| 3 | **Booking List → Filter by PartyPlotId** | `filteredBookingList = bookingList.where((b) => b.partyPlotId == partyPlotId)` then sorted ascending by `bookingDate` | `booking.partyPlotId`, `booking.bookingDate` |
| 4 | **Sort Transactions Ascending** | `filterTransactionList.sort()` by `booking_date` ascending | `transaction.booking_date` |

> **Commented-out filters (partially implemented, needs completion on backend):**
> - Manager name filter on `transaction.related_user_name`
> - Income type filter (`CASH`, `BANK`, `CHEQUE`) on `transaction.amount_type`
> - Status filter (`CREDIT`, `ON_HOLD`) on `transaction.transaction_type` / `transaction.status`

### 5.2 Calculations (Done on Frontend)

| # | Calculation | Formula | Where Used |
|---|-------------|---------|-----------|
| 1 | **Extract unique party plots** | From booking list, fold into a Set by `partyPlotId` — shows unique party plots for dropdown | `income_provider.dart` (line 576–583) |
| 2 | **Sort by booking_date ascending** | Client-side sort for transaction history view | `income_provider.dart` (line 655–672) |

### 5.3 Required Backend Query Parameters

```
GET /booking/transaction
  ?startDate=YYYY-MM-DD
  &endDate=YYYY-MM-DD
  &partyPlotId=<id>
  &partyPlotName=<name>
  &functionDate=YYYY-MM-DD
  &managerId=<id>
  &amountType=CASH|BANK|CHEQUE
  &status=CREDIT|ON_HOLD|PENDING|CANCELLED
  &sortBy=booking_date
  &sortOrder=asc|desc
  &page=1
  &limit=20

GET /user-ledger/transactions/:userId
  ?sortBy=booking_date
  &sortOrder=asc
  &page=1
  &limit=20
```

---

## 6. User Ledger / Transaction History API

**Endpoint:** `GET /user-ledger/ledger`  
**Endpoint (All):** `GET /user-ledger/all/ledger`  
**Endpoint (Billing History):** `GET /user-ledger/party-plot-booking-transactions/:partyPlotId?booking_date=...`  
**Provider File:** `lib/modules/income/provider/income_provider.dart`

### 6.1 Filters (Done on Frontend)

| # | Filter Name | Logic | Field(s) Used |
|---|-------------|-------|---------------|
| 1 | **Sort by Transaction Date** | Bill transaction history sorted ascending by `transaction.transactionDate` | `billTransaction.transaction.transactionDate` (in `BillingProvider.getBillingHistory()`, line 534–538) |

### 6.2 Calculations (Done on Frontend)

> None beyond display formatting. All totals come from the API.

### 6.3 Required Backend Query Parameters

```
GET /user-ledger/party-plot-booking-transactions/:partyPlotId
  ?booking_date=YYYY-MM-DD
  &sortBy=transaction_date
  &sortOrder=asc
  &page=1
  &limit=20
```

---

## 7. Balance Sheet API

**Endpoint:** `GET /billing/profit-loss-analysis`  
**Provider File:** `lib/modules/balance_sheet/provider/balance_sheet_provider.dart`  
**Screen File:** `lib/modules/balance_sheet/screen/balance_screen.dart`

### 7.1 Filters (Done on Frontend)

| # | Filter Name | Logic | Field(s) Used |
|---|-------------|-------|---------------|
| 1 | **Date Range (Financial Year)** | `startDate` and `endDate` passed as query params to API — **already sent to backend** | `financialYearStartDate`, `financialYearEndDate` |

> **Note:** The date filter IS already sent as API params. However, the frontend constructs the financial year dates via `FinancialYear.current()` and there is no additional in-memory filtering.

### 7.2 Calculations (Done on Frontend)

| # | Calculation | Formula | Where Used |
|---|-------------|---------|-----------|
| 1 | **Grand Total Income** (Excel) | `totalIncome += data.income` (summed across all party plots) | `balance_sheet_provider.dart` (line 247, 259) |
| 2 | **Grand Total Expense** (Excel) | `totalExpense += data.totalBilling` | `balance_sheet_provider.dart` (line 260) |
| 3 | **Grand Total Profit** (Excel) | `totalProfit += data.profitLoss` | `balance_sheet_provider.dart` (line 261) |
| 4 | **Grand Profit %** (Excel) | `totalProfitPercent = totalIncome > 0 ? (totalProfit / totalIncome) * 100 : 0` | `balance_sheet_provider.dart` (line 277–279) |
| 5 | **Billing vs Collection Difference** (Screen) | `difference = totalBilling - totalCollection` | `balance_screen.dart` (line 463) |
| 6 | **Payment Breakup** | `cashReceived`, `bankReceived`, `chequeReceived` parsed from `summary.payment_breakup` | `balance_sheet_provider.dart` (line 103–115) — **already from API** |
| 7 | **Total Petty Cash Expense** | Parsed from `summary.total_petty_cash_expense` | `balance_sheet_provider.dart` (line 99–102) — **already from API** |

### 7.3 Notes

- The balance sheet already receives `profit_loss`, `total_gst`, `payment_breakup`, and `total_petty_cash_expense` from the backend summary.
- The Excel export **summary row** grand totals (items 1–4 above) should ideally be returned by the backend as an aggregate field to avoid summing on the client.

### 7.4 Required Backend Additions

```
GET /billing/profit-loss-analysis
  ?startDate=YYYY-MM-DD
  &endDate=YYYY-MM-DD

// Response should include:
{
  "data": {
    "summary": {
      "total_income": <number>,
      "total_billing": <number>,
      "total_profit_loss": <number>,
      "total_profit_loss_percentage": <number>,
      "total_petty_cash_expense": <number>,
      "payment_breakup": {
        "cash_received": <number>,
        "bank_received": <number>,
        "cheque_received": <number>
      }
    },
    "party_plots": [ ... ]
  }
}
```

---

## 8. Summary Table

| API / Module | Frontend Filters to Move to Backend | Frontend Calculations to Move to Backend | Pagination Needed |
|---|---|---|---|
| **Billing List** (`/billing`) | Date range, search (party/user/lead), status (PENDING/CLEAR/EXTRA/CANCELLED), sort | Outstanding amount, computed status, GST back-calculation | ✅ Yes |
| **Cancelled Billing** (`/billing/cancelled`) | Date range, search (party/user/lead), sort descending | Total refundable sum, total waived sum | ✅ Yes |
| **Expense List** (`/petty-cash-expense/`) | Role-based scoping, person name, description, manager name, party plot, head, sub-head, date range | None (amounts are direct) | ✅ Yes |
| **Petty Cash List** (`/petty-cash/get`) | Role-based scoping (ADMIN/MANAGER/ACCOUNTANT), search (name/amount), date range | None | ✅ Yes |
| **Income / Booking Transactions** (`/booking/transaction`) | Party plot, function date, manager, amount type, status, sort | Unique party plot extraction | ✅ Yes |
| **User Transaction History** (`/user-ledger/transactions/:id`) | Sort by booking_date | None | ✅ Yes |
| **Balance Sheet** (`/billing/profit-loss-analysis`) | Date range (already server-side) | Grand totals for Excel (income, billing, profit, profit %) | ❌ Not needed (aggregation) |

---

## 9. Critical Notes for Backend Team

### GST Calculation Inconsistency
In `BillingModel.fromJson()`, when `gst_amount == 0` and `bank_amount > 0`, the frontend **derives** GST as:
```
gstVal = bankAmount - (bankAmount / 1.18)
```
This is a **18% GST back-calculation** from the bank amount. The backend should:
- Always store the actual `gst_amount` correctly.
- Never return `gst_amount = 0` when bank amount indicates GST was charged.

### Outstanding Amount Formula Inconsistency
There is a discrepancy between two parts of the frontend code:
- **Provider `_computeStatus()`:** `outstanding = (totalAmount + gstAmount) - paidAmount`
- **Screen display:** `outstanding = totalAmount - paidAmount` (gstAmount not added)

The backend should return a canonical `outstanding_amount` field and define whether `total_amount` in the DB already includes GST or not.

### Role-Based Filtering
Currently done entirely on the frontend by decoding the JWT token. This is **insecure and unscalable**. The backend must:
- Use the authenticated user's role from the JWT server-side.
- Scope query results accordingly (e.g., `FINANCE_MANAGER` sees only their own expenses).

### Pagination
All list endpoints (`/billing`, `/petty-cash-expense/`, `/petty-cash/get`, `/billing/cancelled`, `/booking/transaction`) currently fetch **all records** and filter/sort in memory. With scale, this will cause:
- High memory usage on the client.
- Slow initial load times.
- Stale data due to in-memory caching.

Each endpoint should return:
```json
{
  "rows": [...],
  "total": 500,
  "page": 1,
  "limit": 20,
  "totalPages": 25
}
```
