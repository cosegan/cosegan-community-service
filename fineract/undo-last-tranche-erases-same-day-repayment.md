# Fineract: Undoing the last tranche erases a repayment made the same day: the loan shows the payment as never made (Loan)

Found via [Cosegan](https://github.com/cosegan) formal/mathematical proof based verification and analysis.

**Effects:** a repayment the client made on the day of the last tranche is erased from the loan when the tranche is undone; if it was paid from savings, the money stays gone from the savings account and is on neither.

**Environment**
- Apache Fineract 1.16.1, the release's own `fineract-provider-1.16.1.jar` (from archive.apache.org, SHA-512 checked), started with `java -jar`
- Java: Eclipse Temurin 25 (Docker image `eclipse-temurin:25-jre`). Database: PostgreSQL 18.3 (Docker image `postgres:18.3`), set up with the release's own scripts
- Default tenant and demo user, default configuration. Changed from the defaults: SSL off and port 8080 (`FINERACT_SERVER_SSL_ENABLED=false`, `FINERACT_SERVER_PORT=8080`), tenant time zone `Asia/Kolkata`. No test profile.

**We verified this on our sandbox only**: a local Fineract 1.16.1 server in Docker, with made-up clients and products. We have not seen it in production. The same logic is on `develop` (commit de5c64036abe, 2026-11-20), in methods that have otherwise changed since 1.16.1; we read it there and did not run it.

### What we are seeing

Fineract refuses to undo a loan's last tranche once the client has repaid after it: with a repayment dated after the tranche, `POST /loans/<loanId>?command=undolastdisbursal` answers 403, "Cannot undo last disbursement after repayments or waivers." A repayment dated the same day as the last tranche does not stop the undo. The undo answers 200 and reverses the repayment along with the tranche, without saying so. The loan then shows the client's payment as never made.

From the first run below (tranches of 600 on 5 October and 400 on 10 October, then a cash repayment of 100 on 10 October):
```
                                    before the undo    after the undo
Loan: principal disbursed           1000.00            600.00
Loan: total repaid                  100.00             0.00
Loan: total outstanding             900.00             600.00    (500.00 if the payment counted)
Loan: repayment of 100, 10 Oct      active             reversed
```

When the repayment is a transfer from the client's savings account (`POST /accounttransfers`), the undo reverses the loan's repayment but not the savings withdrawal and not the transfer. The 100 has left the savings account and is no longer on the loan (second run below: savings balance 0.00, loan outstanding 600.00). Undoing that transfer afterwards answers 403, "Only (non-reversed) transactions of type repayment, waiver, ... can be adjusted", so it cannot be put right through the API.

With the repayment dated 11 October instead, the undo is refused with 403 "Cannot undo last disbursement after repayments or waivers.", as expected.

### What we are expecting

The same answer as for a repayment on a later day: the undo refused with 403 "Cannot undo last disbursement after repayments or waivers." At the least, the client's repayment should not be reversed as a side effect of undoing a tranche. The method that does the reversing has the comment "Reverse only disbursement, accruals, and repayments at disbursal transactions", and an ordinary repayment is none of those.

### How to reproduce

```bash
# Needs bash, curl and jq, and a Fineract 1.16.1 server with the default tenant and the demo user (mifos / password).
BASE=${BASE:-https://localhost:8443/fineract-provider/api/v1}
get()  { echo ">> GET $1" >&2; curl -sk -u mifos:password -H 'Fineract-Platform-TenantId: default' "$BASE$1"; echo; }
post() { echo ">> POST $1" >&2; curl -sk -u mifos:password -H 'Fineract-Platform-TenantId: default' -H 'Content-Type: application/json' -X POST "$BASE$1" -d "$2"; echo; }
D='"locale":"en","dateFormat":"dd MMMM yyyy"'
N=$((RANDOM % 9000 + 1000))   # product names and short names must be unique
PAY_DATE="10 October 2026"     # the day of the last tranche; with "11 October 2026" the undo is refused (403)
summary() { jq -c '{principalDisbursed: .summary.principalDisbursed, totalRepayment: .summary.totalRepayment, totalOutstanding: .summary.totalOutstanding, transactions: [.transactions[] | {id, type: .type.value, date: (.date | map(tostring) | join("-")), amount}]}'; }

# 1. A client, a product that pays out in up to two tranches, and a loan of 1000
CLIENT=$(post /clients '{"officeId":1,"legalFormId":1,"firstname":"Repro","lastname":"R'$N'","active":true,"activationDate":"01 October 2026","submittedOnDate":"01 October 2026",'"$D"'}' | jq -r .resourceId)
LP=$(post /loanproducts '{"name":"Repro tranches '$N'","shortName":"'$N'","currencyCode":"USD","digitsAfterDecimal":2,"inMultiplesOf":0,"principal":1000,"minPrincipal":1,"maxPrincipal":1000000,"numberOfRepayments":4,"repaymentEvery":1,"repaymentFrequencyType":2,"interestRatePerPeriod":0,"interestRateFrequencyType":2,"interestType":0,"interestCalculationPeriodType":0,"amortizationType":1,"inArrearsTolerance":0,"transactionProcessingStrategyCode":"mifos-standard-strategy","accountingRule":1,"daysInMonthType":1,"daysInYearType":1,"isInterestRecalculationEnabled":false,"loanScheduleType":"CUMULATIVE","isEqualAmortization":false,"multiDisburseLoan":true,"maxTrancheCount":2,"outstandingLoanBalance":1000,"disallowExpectedDisbursements":true,'"$D"'}' | jq -r .resourceId)
LOAN=$(post /loans '{"clientId":'$CLIENT',"productId":'$LP',"principal":1000,"loanTermFrequency":4,"loanTermFrequencyType":2,"numberOfRepayments":4,"repaymentEvery":1,"repaymentFrequencyType":2,"interestRatePerPeriod":0,"amortizationType":1,"interestType":0,"interestCalculationPeriodType":0,"transactionProcessingStrategyCode":"mifos-standard-strategy","expectedDisbursementDate":"01 October 2026","submittedOnDate":"01 October 2026","loanType":"individual","maxOutstandingLoanBalance":1000,'"$D"'}' | jq -r .resourceId)
post "/loans/$LOAN?command=approve" '{"approvedOnDate":"01 October 2026","approvedLoanAmount":1000,"expectedDisbursementDate":"01 October 2026",'"$D"'}'

# 2. Two tranches: 600 on 5 October, 400 on 10 October
post "/loans/$LOAN?command=disburse" '{"actualDisbursementDate":"05 October 2026","transactionAmount":600,'"$D"'}'
post "/loans/$LOAN?command=disburse" '{"actualDisbursementDate":"10 October 2026","transactionAmount":400,'"$D"'}'

# 3. The client repays 100 in cash
REPAYMENT=$(post "/loans/$LOAN/transactions?command=repayment" '{"transactionDate":"'"$PAY_DATE"'","transactionAmount":100,'"$D"'}' | jq -r .resourceId)
get "/loans/$LOAN?associations=transactions" | summary

# 4. Undo the last tranche: 200 (403 when the repayment is on a later day)
post "/loans/$LOAN?command=undolastdisbursal" '{"note":"undo the 400 tranche"}'

# 5. The repayment is gone: the loan owes 600, not 500
get "/loans/$LOAN?associations=transactions" | summary
get "/loans/$LOAN/transactions/$REPAYMENT" | jq -c '{id, type: .type.value, amount, reversedOnDate}'
```

The same with the repayment made by a transfer from the client's savings account. This continues in the same shell (it uses `get`, `post`, `summary`, `D`, `N`, `PAY_DATE`, `CLIENT` and `LP` from above):

```bash
# 6. A savings account (USD, no interest) with 100 in it, and a second loan of 1000 with the same two tranches
SP=$(post /savingsproducts '{"name":"Repro savings '$N'","shortName":"'$N'","currencyCode":"USD","digitsAfterDecimal":2,"inMultiplesOf":0,"nominalAnnualInterestRate":0,"interestCompoundingPeriodType":1,"interestPostingPeriodType":4,"interestCalculationType":1,"interestCalculationDaysInYearType":365,"accountingRule":1,"locale":"en"}' | jq -r .resourceId)
SAVINGS=$(post /savingsaccounts '{"clientId":'$CLIENT',"productId":'$SP',"submittedOnDate":"01 October 2026",'"$D"'}' | jq -r .resourceId)
post "/savingsaccounts/$SAVINGS?command=approve" '{"approvedOnDate":"01 October 2026",'"$D"'}'
post "/savingsaccounts/$SAVINGS?command=activate" '{"activatedOnDate":"01 October 2026",'"$D"'}'
post "/savingsaccounts/$SAVINGS/transactions?command=deposit" '{"transactionDate":"01 October 2026","transactionAmount":100,"paymentTypeId":1,'"$D"'}'
LOAN2=$(post /loans '{"clientId":'$CLIENT',"productId":'$LP',"principal":1000,"loanTermFrequency":4,"loanTermFrequencyType":2,"numberOfRepayments":4,"repaymentEvery":1,"repaymentFrequencyType":2,"interestRatePerPeriod":0,"amortizationType":1,"interestType":0,"interestCalculationPeriodType":0,"transactionProcessingStrategyCode":"mifos-standard-strategy","expectedDisbursementDate":"01 October 2026","submittedOnDate":"01 October 2026","loanType":"individual","maxOutstandingLoanBalance":1000,'"$D"'}' | jq -r .resourceId)
post "/loans/$LOAN2?command=approve" '{"approvedOnDate":"01 October 2026","approvedLoanAmount":1000,"expectedDisbursementDate":"01 October 2026",'"$D"'}'
post "/loans/$LOAN2?command=disburse" '{"actualDisbursementDate":"05 October 2026","transactionAmount":600,'"$D"'}'
post "/loans/$LOAN2?command=disburse" '{"actualDisbursementDate":"10 October 2026","transactionAmount":400,'"$D"'}'

# 7. The client repays 100 from savings, on the day of the last tranche
post /accounttransfers '{"fromOfficeId":1,"fromClientId":'$CLIENT',"fromAccountType":2,"fromAccountId":'$SAVINGS',"toOfficeId":1,"toClientId":'$CLIENT',"toAccountType":1,"toAccountId":'$LOAN2',"transferDate":"'"$PAY_DATE"'","transferAmount":100,"transferDescription":"repayment from savings",'"$D"'}'
REPAYMENT2=$(get "/loans/$LOAN2?associations=transactions" | jq '.transactions[] | select(.type.value == "Repayment") | .id')
TRANSFER=$(get "/loans/$LOAN2/transactions/$REPAYMENT2" | jq '.transfer.id')   # the id that GET /accounttransfers/<id> takes
get "/loans/$LOAN2?associations=transactions" | summary

# 8. Undo the last tranche: 200
post "/loans/$LOAN2?command=undolastdisbursal" '{"note":"undo the 400 tranche"}'

# 9. The loan's repayment is reversed; the savings withdrawal and the transfer are not
get "/loans/$LOAN2?associations=transactions" | summary
get "/loans/$LOAN2/transactions/$REPAYMENT2" | jq -c '{id, type: .type.value, amount, reversedOnDate, transfer: {id: .transfer.id, reversed: .transfer.reversed}}'
get "/savingsaccounts/$SAVINGS?associations=transactions" | jq -c '{accountBalance: .summary.accountBalance, transactions: [.transactions[] | {id, type: .transactionType.value, amount, reversed, transferId: .transfer.id}]}'
get "/accounttransfers/$TRANSFER" | jq -c '{id, transferAmount, reversed}'
post "/accounttransfers/$TRANSFER?command=undo" '{}' | jq -c '{httpStatusCode, message: .errors[0].developerMessage}'
```

### Where in the code

`LoanWritePlatformServiceJpaRepositoryImpl` (fineract-provider), line numbers as in 1.16.1:
- `undoLastDisbursal` (line 3443) refuses the undo, in the loop at lines 3452 to 3461, only when a repayment, waiver or charge payment is dated after the last disbursement: `DateUtils.isBefore(lastTransactionDate, previousTransaction.getTransactionDate())`. A repayment on the same date passes.
- `reverseExistingTransactionsTillLastDisbursal` (line 3502; the loop at lines 3528 to 3536) then reverses every transaction dated on or after the last disbursement, entered after it, for which `LoanTransaction.isAllowTypeTransactionAtTheTimeOfLastUndo()` is true. That method (fineract-loan, `LoanTransaction.java` lines 983 to 985) includes `isRepayment()`, so the same-day repayment is reversed. Its only caller is line 3531.
- `undoLastLoanDisbursal` (line 2652) does not call the account transfer service, so a repayment made by transfer loses only its loan side.

On `develop` (de5c64036abe) `undoLastLoanDisbursal`, `undoLastDisbursal` and `reverseExistingTransactionsTillLastDisbursal` start at lines 2076, 2664 and 2718, and `isAllowTypeTransactionAtTheTimeOfLastUndo` at `LoanTransaction.java` line 987.

### Possible fix

Refuse the undo when a repayment, waiver or charge payment on the same date as the last disbursement was entered after it (a higher id), as for a later date; and take `isRepayment()` out of `isAllowTypeTransactionAtTheTimeOfLastUndo()`, so the method reverses what its comment says. We have not built or tested this.

---

Posted to the project: https://issues.apache.org/jira/browse/FINERACT-3321
