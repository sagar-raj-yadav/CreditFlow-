# Loan Management System

## 1. Models

### User Model
- **User Details:** Name, Email, Aadhar ID, Annual Income, Credit Score.
- **Credit Score Calculation:** Based on transaction history.

### Loan Model
- Stores loan details: User ID, Loan Amount, Interest Rate, Tenure, EMI Schedule, and Outstanding Balance.

### Billing Model
- Stores billing cycles with interest accrued and minimum due amount.
- Tracks due dates, previous pending payments, and interest calculations.

### Payment Model
- Records loan repayments and maps them to billing cycles.
- Ensures payments clear past dues before new dues.

### Transaction Model (from CSV)
- Stores transaction history (CREDIT/DEBIT) for credit score calculation.

## 2. Process Flow

### (A) User Registration & Credit Score Calculation
1. User registers via `/api/register-user/`.
2. A Celery task is triggered to:
   - Read transactions from CSV based on Aadhar ID.
   - Compute total balance (CREDIT - DEBIT).
   - Assign a credit score (range 300-900) based on balance.
   - Store the credit score in the database.

### (B) Loan Application & Disbursement
1. User applies for a loan via `/api/apply-loan/`.
2. System validates:
   - Credit score >= 450.
   - Annual income >= Rs. 1,50,000.
   - Loan amount <= Rs. 50,000.
3. If approved:
   - A loan record is created.
   - EMI schedule is generated.
   - Interest accrual starts from Day 1.

### (C) EMI Calculation
1. Interest accrues daily:
   - `daily_apr_accrued = round(apr / 365, 3)`
2. Minimum Due Amount:
   - `(Principal Balance * 3%) + (Total Daily Interest for the month)`
3. First EMI is scheduled 30 days after loan disbursal.

### (D) Repayment & Payment Handling
1. User makes payments via `/api/make-payment/`.
2. System checks:
   - No past EMI is unpaid.
   - Payment is not duplicated.
   - Amount matches EMI (or recalculates EMI).
3. Loan balance is updated atomically to prevent race conditions.

### (E) Billing Process
1. Cron job runs daily to check users due for billing.
2. For each due user:
   - Interest is computed for the billing cycle.
   - Minimum due is calculated and recorded.
   - Due date is set to 15 days from the billing date.
3. Outstanding past dues are carried forward.

### (F) Fetching Statements & Future Dues
1. Users retrieve past transactions & future EMIs via `/api/get-statement/`.
2. Response includes:
   - Past transactions with date, principal, interest, and amount paid.
   - Upcoming EMIs with date and amount due.

## 3. Challenges & Solutions

| Challenge | Solution |
|-----------|----------|
| Handling large transaction history for credit score | Use batch processing in Celery to compute scores asynchronously. |
| Preventing race conditions in payments | Use atomic transactions and locking mechanisms to update balances. |
| Ensuring scalability for billing | Use Cron jobs and batch updates instead of real-time computations. |
| Avoiding performance issues due to N+1 queries | Optimize Django ORM queries with `prefetch_related` and `select_related`. |
| Recalculating EMI if payment is incorrect | Implement adjustment logic to update the EMI schedule dynamically. |

## 4. Authentication
For secure access, we implement JWT authentication and verify Aadhar ID before processing requests.

### (A) Using JWT for Secure Authentication
- JWT (JSON Web Token) provides stateless authentication.
- User logs in → JWT is issued → Every API request must include JWT in headers.
- Generate JWT on login → Verify JWT before accessing APIs.

## 5. Database Design & Query Optimization

### Entities & Relationships

#### User Model
- Stores user details, including Aadhar ID, Name, Email, Annual Income, and Credit Score.
- Aadhar ID is unique and indexed for fast lookups.

#### Loan Model
- Linked to the User model via a ForeignKey.
- Stores Loan Amount, Interest Rate, Tenure, EMI details, and Loan Status.

#### Payment Model
- Linked to the Loan model via a ForeignKey.
- Stores Transaction ID, Amount Paid, Due Date, and Payment Status.

### Indexing
1. **Indexing on Aadhar ID:**
   - Used for user verification and transaction lookups, improving query performance.
2. **Bulk Queries Instead of N+1 Problems:**
   - Avoid fetching one payment record at a time (N+1 problem).
   - Use `select_related` and `prefetch_related` for optimized queries.

## 6. Celery Task for Credit Score Calculation
Since calculating a credit score is computationally expensive, we process it asynchronously using Celery with Redis as a broker.

### Why Use Celery?
- The credit score calculation might require analyzing past transactions.
- This can take time, so we run it asynchronously instead of making the user wait.

## 7. Atomic Transactions
Since financial transactions must be consistent, Django’s atomic transactions ensure that updates do not fail midway.

### Why Use Atomic Transactions?
- If a payment is processed but the database update fails, the system may show incorrect balances.
- **Solution:** Wrap payment updates in an atomic transaction.

### Optimistic Locking for Concurrency Control
- `select_for_update()` locks the row to prevent race conditions.
- Ensures that two payments cannot be processed at the same time, avoiding double deductions.

## 8. Conclusion
✅ **Fast:** Uses Celery for asynchronous processing.
✅ **Secure:** JWT authentication & optimistic locking.
✅ **Scalable:** Batch processing & index optimization.
✅ **Reliable:** Atomic transactions prevent financial inconsistencies.

