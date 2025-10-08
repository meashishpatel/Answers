# Answers

## Answer1

The transaction flow shows that after the funds were successfully debited by Axis Bank at 15:10:50, NPCI immediately requested Yes Bank to credit the receiver at 15:10:50. Although Yes Bank's internal database shows the credit was processed almost instantly (at 15:10:51 and 15:10:52), it failed to send a success confirmation back to NPCI.

The logs show that Yes Bank only sent the credit/resp API call confirming the successful transaction to NPCI at 17:10:52, a full two hours after the transaction was initiated. This delay in communication from Yes Bank was the root cause of the entire issue, as all other entities were waiting for this final confirmation.

## Answer2

Yes, the funds were both deducted from the sender and credited to the receiver.

* **Fund Deduction:** The logs confirm that Axis Bank successfully debited the sender's account. The database query `UPDATE RemitterCBS SET amount = amount - 1.00` was executed at 15:10:49. Axis Bank then sent a SUCCESS confirmation to NPCI at 15:10:50. To verify this, one would check the `debitFunds` API calls and the `RemitterCBS` and `RemitterPassBook` tables at Axis Bank's end.

* **Fund Credit:** The funds were credited to the receiver's account by Yes Bank. The database query `UPDATE BeneficiaryCBS SET amount = amount + 1.00` was executed at 15:10:51. To verify this, one would check the `creditFunds` API calls and the `BeneficiaryCBS` and `BeneficiaryPassBook` tables at Yes Bank's end.

## Answer3

Paytm was attempting to get the transaction status by repeatedly calling the `statusCheck` API to NPCI. The logs show multiple outgoing GET requests from Paytm (PTY) to NPCI's status endpoint, for instance at 15:30:00 and 16:55:00.

The entity providing the latest available transaction update to Paytm was NPCI. However, NPCI itself was waiting for the final confirmation from Yes Bank. Therefore, until 17:10:52, NPCI could only respond with a PENDING status, as it hadn't received the success callback from Yes Bank.

## Answer4

### 1. Verify the Final Transaction Status

Before making any changes, you must confirm the transaction's true, final state. The most critical step is to query the central authority for all UPI payments.

* **Query NPCI:** Use an internal tool to check the status of the transaction using its unique `npciRequestId` (`HST<HIDDEN>9910`) directly with NPCI. NPCI is the authoritative source of truth for the entire transaction lifecycle.
  
* **Cross-Verify with Banks (Optional):** As a secondary check, you can verify that the funds were indeed debited from Axis Bank and credited to Yes Bank, but the NPCI status is the primary trigger for this manual update.

### 2. Obtain Authorization

A manual database change requires formal approval.

* **Log the Evidence:** Document the confirmation of success from NPCI.
  
* **Request Approval:** Present the evidence to a manager or team lead to get explicit approval for the manual intervention. This creates an audit trail and ensures accountability.

### 3. Execute the Database Update

Once authorized, a designated engineer will perform the update on Paytm's database only. You must never alter bank-side ledgers.

* **Identify the Record:** Locate the transaction in Paytm's internal tables using the `npciRequestId`.
  
* **Run Update Query:** Execute SQL commands to change the status from PENDING to SUCCESS. Based on the logs, this would involve updating tables like `TRANSACTION_LS_202406` and `customerTransactionDetails`.

```sql
-- Update the main transaction log
UPDATE TRANSACTION_LS_202406
SET status = 'SUCCESS', updated_at = NOW()
WHERE npciRequestId = 'HST<HIDDEN>9910';

-- Update the customer-facing details table
UPDATE customerTransactionDetails
SET status = 'SUCCESS', updated_at = NOW()
WHERE npciRequestId = 'HST<HIDDEN>9910';
```




## Answer 5
The latency for the statusCheck API call with requestId: avd3-XXXX-ae3 was 33 seconds.

This is calculated by comparing the timestamp in the request headers (16:55:00) with the timestamp in the response headers (16:55:33) for that specific requestId in the log file.

For reference: {"x-request-id":"uisu-XXXX-8uj","x-client-id":"<HIDDEN>","content-type":"application/json","x-app-name":"juspay","x-timestamp":"2024-06-05 16:55:33"}

## Answer 6
When a user initiates a ₹500 UPI payment, NPCI performs a series of database operations to record and track the transaction lifecycle. Based on the observed patterns in the log file, the following queries represent the typical flow at the NPCI end:

1. Initial Insert — Transaction Creation
Upon receiving the transaction request from the PSP (e.g., Paytm), NPCI creates a new record marking both debit and credit statuses as PENDING.
```sql
INSERT INTO newTransactions 
    (npciRequestId, amount, payerVPA, payeeVPA, debitStatus, creditStatus, created_at)
VALUES 
    ('<npciRequestId>', 500.00, 'payer@axis', 'payee@yes', 'PENDING', 'PENDING', CURRENT_TIMESTAMP);
```

2. Debit Status Update — Confirmation from Payer’s Bank
Once NPCI receives a success acknowledgment from the payer’s bank (e.g., Axis Bank), it updates the debit status to SUCCESS.

```sql
UPDATE newTransactions
SET debitStatus = 'SUCCESS', debitUpdatedAt = CURRENT_TIMESTAMP
WHERE npciRequestId = '<npciRequestId>';
```

3. UPDATE newTransactions
SET debitStatus = 'SUCCESS', debitUpdatedAt = CURRENT_TIMESTAMP
WHERE npciRequestId = '<npciRequestId>';

```sql
UPDATE newTransactions
SET creditStatus = 'SUCCESS', creditUpdatedAt = CURRENT_TIMESTAMP
WHERE npciRequestId = '<npciRequestId>';
```

4. Status Check — Queried by Participant Applications
When an app (like Paytm or Google Pay) performs a status check, NPCI retrieves the current debit and credit status for that transaction.

```sql
SELECT debitStatus, creditStatus 
FROM newTransactions 
WHERE npciRequestId = '<npciRequestId>';
```

## Answer 7
* Synchronous APIs: These calls require an immediate response for the flow to continue.

    * resolveAddress: NPCI must wait for GPay to return the receiver's bank details before it can proceed.

    * statusCheck: Paytm makes this call and waits for an immediate response from NPCI regarding the current transaction state.

* Asynchronous APIs: These calls are initiated, and the system moves on after receiving an acknowledgement (like "Processing"), without waiting for the final result. The final result is communicated later via a separate callback.

    * makePayment: The primary payment initiation from Paytm to NPCI is asynchronous. Paytm gets a PENDING status and waits for a final success/failure notification later.

    * debitFunds / creditFunds: The calls from NPCI to the banks are asynchronous. NPCI receives an initial PENDING response and relies on separate callback APIs (debit/resp, credit/resp) from the banks to get the final status.



## Answer 8
The customer saw the transaction as "Pending" for over an hour because the final success confirmation was delayed. The entity that contributed to this delay was Yes Bank.


Although Yes Bank processed the credit internally at 15:10:51, it failed to send the success callback to NPCI until 17:10:52. Because NPCI did not have this final confirmation, it continued to report the transaction status as PENDING to Paytm during its periodic status checks.


## Answer 9
The final status of the transaction is SUCCESS.

I came to this conclusion by following the logs to the very end of the transaction lifecycle. The key events confirming the success are:

1. Yes Bank sends a credit/resp call to NPCI with "transactionStatus":"SUCCESS" at 17:10:52.

2. NPCI updates its database, setting creditStatus = 'SUCCESS' at 17:10:53.

3. NPCI sends a final notification to Paytm with "result":"SUCCESS" at 17:10:54.


4. Paytm updates its internal database tables to reflect the SUCCESS status.


## Answer 10
The receiver's claim that they received ₹2 is true.

My analysis of the logs for Yes Bank's database reveals a critical issue. There are two separate UPDATE queries executed against the BeneficiaryCBS table, one right after the other:


    1. UPDATE BeneficiaryCBS SET amount = amount + 1.00 ... at 15:10:51.


    2. UPDATE BeneficiaryCBS SET amount = amount + 1.00 ... at 15:10:52.

This indicates a bug, likely a race condition or a lack of idempotency in the credit processing logic at Yes Bank's end, which caused the customer's account to be credited twice for a single incoming transaction.

## Answer 11
Yes, delayed transaction confirmations and reconciliation issues are common challenges in distributed payment systems. Here is how such issues can be prevented in the future:

    * Stricter SLAs: Enforce and monitor stricter Service Level Agreements (SLAs) with partner banks for the turnaround time of asynchronous callbacks. Breaches should trigger alerts and potentially have financial penalties.

    * Proactive Monitoring & Alerting: Implement a monitoring system that tracks the age of transactions. If a transaction remains in a PENDING state beyond a reasonable threshold (e.g., 5 minutes), an automatic alert should be fired to the engineering team for proactive investigation.

    * Automated Reconciliation: Instead of relying solely on delayed callbacks, build an automated reconciliation service that periodically polls NPCI for the status of pending transactions and updates the system accordingly. This ensures that the user-facing status is accurate even if a partner bank fails to send a timely response.


## Answer 12
### 🏦 Delays at the Bank's End
The most common delays originate from the bank's internal systems.

* Core Banking System (CBS) Delays: A bank's internal systems might be slow, under heavy load, or may process certain transactions in batches rather than in real-time.

* Callback Service Outage: The specific service at the bank responsible for sending the final success confirmation back to NPCI might be down or malfunctioning, even if the customer's account was successfully credited.

* Application Logic Bugs: Flaws in the bank's software can lead to incorrect behavior, such as dropping confirmation messages or, as seen with the double-credit issue, processing payments incorrectly.

### 🌐 Delays in Communication and Infrastructure
The issue can also occur while the confirmation message is in transit between systems.

* Network Latency or Failure: The connection between the bank and NPCI can be slow or unreliable, causing the callback message to be delayed or lost entirely.

* Message Queue Issues: In modern systems, messages are passed through queues. If a downstream service is slow, backpressure can build up in these queues, delaying all subsequent messages.

* Infrastructure Failures: A server or network node could be down, preventing the message from being sent or received.

### 🏢 Delays at the Central Switch (NPCI)
Even if the bank sends a timely confirmation, the central switch can be a source of delay.

* Ingestion and Database Lag: NPCI's systems might be slow to process an incoming confirmation message or experience delays in updating their own databases, especially during peak loads.

### 🛠️ Operational and Data Issues
Finally, some delays are caused by issues that require human intervention.

* Malformed Data: The callback from the bank could be malformed or misrouted, causing it to fail processing and require manual correction.

* Manual Review: A transaction might be flagged for a fraud check or operational review, pausing the automated flow until it is manually approved.

## Answer 13
Immediate Actions
* Verify Status with NPCI: The immediate action is to use an internal tool to check the definitive status of the transaction with NPCI.

* Manually Reconcile: Once confirmed as SUCCESS by NPCI, manually update the transaction status in the Paytm database to SUCCESS to resolve the issue for the customer.

* Raise Incident with Yes Bank: Immediately open a high-priority incident with the technical team at Yes Bank, providing the npciRequestId and logs. This is crucial to get them to investigate the two-hour callback delay and the critical double-credit bug.

Immediate Response to Customer
A clear and empathetic response is key.

"Hi [Customer Name],

Thank you for bringing this to our attention. We have investigated your transaction with ID HST<HIDDEN>9910.

We can confirm that the payment of ₹1 was successfully credited to your friend's account. The 'Pending' status you observed was due to a temporary delay in receiving the final confirmation from the receiver's bank. This has now been resolved, and the correct status should be reflected in your app.

We sincerely apologize for the confusion and appreciate your patience.

Thank you,
Juspay Support"


Longer term: fix automation so Paytm auto-marks success when both bank passbooks show credit or when NPCI confirms within configured SLA; add compensating/rollback flows if end outcome is inconsistent.