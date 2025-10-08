# Answers

## Answer1

The transaction flow shows that after the funds were successfully debited by Axis Bank at 15:10:50, NPCI immediately requested Yes Bank to credit the receiver at 15:10:50. Although Yes Bank's internal database shows the credit was processed almost instantly (at 15:10:51 and 15:10:52), it failed to send a success confirmation back to NPCI.

The logs show that Yes Bank only sent the credit/resp API call confirming the successful transaction to NPCI at 17:10:52, a full two hours after the transaction was initiated. This delay in communication from Yes Bank was the root cause of the entire issue, as all other entities were waiting for this final confirmation.

## Answer2

Yes, the funds were both deducted from the sender and credited to the receiver.

- **Fund Deduction:** The logs confirm that Axis Bank successfully debited the sender's account. The database query `UPDATE RemitterCBS SET amount = amount - 1.00` was executed at 15:10:49. Axis Bank then sent a SUCCESS confirmation to NPCI at 15:10:50. To verify this, one would check the `debitFunds` API calls and the `RemitterCBS` and `RemitterPassBook` tables at Axis Bank's end.

- **Fund Credit:** The funds were credited to the receiver's account by Yes Bank. The database query `UPDATE BeneficiaryCBS SET amount = amount + 1.00` was executed at 15:10:51. To verify this, one would check the `creditFunds` API calls and the `BeneficiaryCBS` and `BeneficiaryPassBook` tables at Yes Bank's end.

## Answer3

Paytm was attempting to get the transaction status by repeatedly calling the `statusCheck` API to NPCI. The logs show multiple outgoing GET requests from Paytm (PTY) to NPCI's status endpoint, for instance at 15:30:00 and 16:55:00.

The entity providing the latest available transaction update to Paytm was NPCI. However, NPCI itself was waiting for the final confirmation from Yes Bank. Therefore, until 17:10:52, NPCI could only respond with a PENDING status, as it hadn't received the success callback from Yes Bank.

## Answer4

### 1. Verify the Final Transaction Status

Before making any changes, you must confirm the transaction's true, final state. The most critical step is to query the central authority for all UPI payments.

- **Query NPCI:** Use an internal tool to check the status of the transaction using its unique `npciRequestId` (`HST<HIDDEN>9910`) directly with NPCI. NPCI is the authoritative source of truth for the entire transaction lifecycle.
  
- **Cross-Verify with Banks (Optional):** As a secondary check, you can verify that the funds were indeed debited from Axis Bank and credited to Yes Bank, but the NPCI status is the primary trigger for this manual update.

### 2. Obtain Authorization

A manual database change requires formal approval.

- **Log the Evidence:** Document the confirmation of success from NPCI.
  
- **Request Approval:** Present the evidence to a manager or team lead to get explicit approval for the manual intervention. This creates an audit trail and ensures accountability.

### 3. Execute the Database Update

Once authorized, a designated engineer will perform the update on Paytm's database only. You must never alter bank-side ledgers.

- **Identify the Record:** Locate the transaction in Paytm's internal tables using the `npciRequestId`.
  
- **Run Update Query:** Execute SQL commands to change the status from PENDING to SUCCESS. Based on the logs, this would involve updating tables like `TRANSACTION_LS_202406` and `customerTransactionDetails`.

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

## Answer 5
When a user initiates a ₹500 UPI payment, NPCI performs a series of database operations to record and track the transaction lifecycle. Based on the observed patterns in the log file, the following queries represent the typical flow at the NPCI end:

Initial Insert — Transaction Creation
Upon receiving the transaction request from the PSP (e.g., Paytm), NPCI creates a new record marking both debit and credit statuses as PENDING.
```sql
INSERT INTO newTransactions 
    (npciRequestId, amount, payerVPA, payeeVPA, debitStatus, creditStatus, created_at)
VALUES 
    ('<npciRequestId>', 500.00, 'payer@axis', 'payee@yes', 'PENDING', 'PENDING', CURRENT_TIMESTAMP);
```

