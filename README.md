## MODEL
1.user model
a.user details:-name, email, Aadhar ID, annual income, credit score
b.Credit score is calculated based on transaction history.

2.Loan Model
a.Stores loan details: user ID, loan amount, interest rate, tenure, EMI schedule, and outstanding balance.

3.Billing Model
a.Stores billing cycles with interest accrued and min due amount.
b.Tracks due dates, previous pending payments, and interest calculation.

4.Payment Model
a.Records loan repayments and maps them to billing cycles.
b.Ensures payments clear past dues before new dues.

5.Transaction Model (from CSV)
a.Stores transaction history (CREDIT/DEBIT) for credit score calculation.

## Process Flow
(A) User Registration & Credit Score Calculation
i.User registers via /api/register-user/.
ii.A Celery task is triggered to:
iii.Read transactions from CSV based on Aadhar ID.
iv.Compute total balance (CREDIT - DEBIT).
v.Assign credit score (range 300-900) based on balance.
vi.Store credit score in the database.

(B)Loan Application & Disbursement
i.User applies for a loan via /api/apply-loan/.
    The system validates:
    Credit score >= 450.
    Annual income >= Rs. 1,50,000.
    Loan amount <= Rs. 5000.
ii.If approved, a loan record is created, and:
    EMI schedule is generated.
    Interest accrual starts from Day 1

(C) EMI Calculation
i.Interest accrues daily as per:
    daily_apr_accrued = round(apr / 365, 3)
ii.Min due = (Principal Balance * 3%) + (Total Daily Interest for the month).
iii.The first EMI is scheduled 30 days after loan disbursal.

(D) Repayment & Payment Handling
i.User makes payments via /api/make-payment/.
ii.System checks:
     No past EMI is unpaid.
     Payment is not duplicated.
     Amount matches EMI (or recalculates EMI).
iii.Loan balance is updated atomically to prevent race conditions.

(E) Billing Process
i.Cron Job runs daily to check users due for billing.
ii.For each due user:
   Interest is computed for the billing cycle.
   Min due is calculated and recorded.
   Due date is set to 15 days from the billing date.
iii.If a user has outstanding past dues, they are carried forward.

(F) Fetching Statements & Future Dues
i.Users retrieve past transactions & future EMIs via /api/get-statement/.
ii.Response includes:
   Past transactions with date, principal, interest, and amount paid.
   Upcoming EMIs with date and amount due.


## 5. Challenges & Solutions

Challenge	                                                      Solution
i.Handling large transaction history for credit score	          Use batch   processing in Celery to compute scores asynchronously.

ii.Preventing race conditions in payments	                      Use atomic transactions and locking mechanisms to update balances.

iii.Ensuring scalability for billing	                          Use Cron jobs and batch updates instead of real-time computations.

iv.Avoiding performance issues due to N+1 queries	               Optimize Django ORM queries with prefetch_related and select_related.

v.Recalculating EMI if payment is incorrect	                       Implement adjustment logic to update EMI schedule dynamically.

### 6.Authentication 
For secure access, we implement JWT authentication and verify Aadhar ID before processing requests.

i. Using JWT for Secure Authentication
JWT (JSON Web Token) provides stateless authentication.
User logs in → JWT is issued->Every API request must include JWT in headers

Generate JWT on login->Verify JWT before accessing APIs

### 7. Database Design & Query Optimization
=>Make ForeignKey relationships between users,Loan,payments.

# Entities & Relationships
a.user model
->Stores user details, including Aadhar ID, Name, Email, Annual Income, and Credit Score.
->Aadhar ID is unique and indexed for fast lookups.

b.Loan Model
->LOan model is Linked to the User model via a ForeignKey.
->Stores Loan Amount, Interest Rate, Tenure, EMI details, and Loan Status.

c.Payment Model
->Payment Model Linked to the Loan model via a ForeignKey.
->Stores Transaction ID, Amount Paid, Due Date, and Payment Status.

# Indexing
a.Indexing on Aadhar ID
->Since Aadhar ID is used for user verification and transaction lookups, we ->index it to improve query performance.

b.Bulk Queries Instead of N+1 Problems
->Avoid fetching one payment record at a time (N+1 problem).
->Use select_related and prefetch_related for optimized queries.

### Celery Task for Credit Score Calculation
Since calculating a credit score is computationally expensive, we will process it asynchronously using Celery with Redis as a broker.

i. Why Use Celery?
->The credit score calculation might require analyzing past transactions.
->This can take time, so we run it asynchronously instead of making the user wait.

### Atomic Transactions
Since financial transactions must be consistent, Django’s atomic transactions ensure that updates do not fail midway.

i.Why Use Atomic Transactions?
->If a payment is processed but the database update fails, the system may show incorrect balances.
->Solution: Wrap payment updates in an atomic transaction.

ii.Optimistic Locking for Concurrency Control
->select_for_update() locks the row to prevent race conditions.
->Ensures that two payments cannot be processed at the same time, avoiding double deductions.


### conclusion
✅ Fast (using Celery for async processing).
✅ Secure (JWT authentication & optimistic locking).
✅ Scalable (batch processing & index optimization).
✅ Reliable (atomic transactions prevent financial inconsistencies).