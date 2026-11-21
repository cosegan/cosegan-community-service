# Fineract: Undoing the last tranche leaves it in the linked savings account: the client keeps money the loan no longer shows as lent (Loan, Savings)

Found via [Cosegan](https://github.com/cosegan) formal/mathematical proof based verification and analysis.

**Effects:** undoing a loan's last tranche leaves that money in the client's savings account while the loan stops showing it as lent, so it is never asked back; neither side can be undone afterwards.

**Environment**
- Apache Fineract 1.16.1, the release's own `fineract-provider-1.16.1.jar` (from archive.apache.org, SHA-512 checked), started with `java -jar`
- Java: Eclipse Temurin 25 (Docker image `eclipse-temurin:25-jre`). Database: PostgreSQL 18.3 (Docker image `postgres:18.3`), set up with the release's own scripts
- Default tenant and demo user, default configuration. Changed from the defaults: SSL off and port 8080 (`FINERACT_SERVER_SSL_ENABLED=false`, `FINERACT_SERVER_PORT=8080`), tenant time zone `Asia/Kolkata`. No test profile.

**We verified this on our sandbox only**: a local Fineract 1.16.1 server in Docker, with made-up clients and products. We have not seen it in production. The same logic is on `develop` (commit de5c64036abe, 2026-11-20), in methods that have otherwise changed since 1.16.1; we read it there and did not run it.

### What we are seeing

A loan that pays out in tranches is linked to the client's savings account, and both tranches are paid into that account (`command=disburseToSavings`): 600, then 400. Undoing the last tranche (`POST /loans/<loanId>?command=undolastdisbursal`) reverses the loan's 400 disbursement, but not the 400 deposit in the savings account and not the account transfer that links the two. The answer is 200 and says nothing about the savings side.

The client keeps the 400 in savings, free to withdraw, while the loan no longer shows it as lent, so the loan will never ask for it back.

From the run below:
```
                                    before the undo    after the undo
Loan: principal disbursed           1000.00            600.00
Loan: total outstanding             1000.00            600.00
Loan: disbursement of 400, 10 Oct   active             reversed
Savings: account balance            1000.00            1000.00
Savings: deposit of 400, 10 Oct     active             active
Account transfer of the 400         active             active
```

Read by id, the reversed 400 disbursement has `reversedOnDate` set and still names its account transfer, and that transfer says `"reversed": false`.

Afterwards neither side can be undone through the API:
- `POST /accounttransfers/<transferId>?command=undo` answers 501, "Undo Loan to Savings Account Transfer is not implemented".
- `POST /savingsaccounts/<savingsId>/transactions/<transactionId>?command=undo` answers 503, "Savings account transaction:<id> update not allowed as it involves in account transfer".

With cash accounting on (a separate run of ours with the same steps, GL accounts on both products, and the "liability transfer" financial activity mapped to a GL account), the ledger after the undo has the loan portfolio at 600.00, the savings control account still at 1,000.00 owed to the client, and the 400.00 difference left as a debit on the liability transfer account.

The full undo (`command=undodisbursal`) on the same kind of loan reverses both deposits and both transfers, and the savings balance goes back to 0.00, as expected.

### What we are expecting

Undoing the last tranche should undo all of it, as the full undo does: reverse the 400 deposit and its account transfer together with the loan's disbursement. If the savings side cannot be reversed (for example, because the client has already withdrawn the money), the undo should be refused rather than leave the two accounts disagreeing.

### How to reproduce

```bash
# Needs bash, curl and jq, and a Fineract 1.16.1 server with the default tenant and the demo user (mifos / password).
BASE=${BASE:-https://localhost:8443/fineract-provider/api/v1}
get()  { echo ">> GET $1" >&2; curl -sk -u mifos:password -H 'Fineract-Platform-TenantId: default' "$BASE$1"; echo; }
post() { echo ">> POST $1" >&2; curl -sk -u mifos:password -H 'Fineract-Platform-TenantId: default' -H 'Content-Type: application/json' -X POST "$BASE$1" -d "$2"; echo; }
D='"locale":"en","dateFormat":"dd MMMM yyyy"'
N=$((RANDOM % 9000 + 1000))   # product names and short names must be unique
loan_view()    { jq -c '{principalDisbursed: .summary.principalDisbursed, totalOutstanding: .summary.totalOutstanding, transactions: [.transactions[] | {id, type: .type.value, amount}]}'; }
savings_view() { jq -c '{accountBalance: .summary.accountBalance, transactions: [.transactions[] | {id, type: .transactionType.value, amount, reversed, transferId: .transfer.id}]}'; }

# 1. A client and a savings account (USD, no interest), opened on 1 October 2026
CLIENT=$(post /clients '{"officeId":1,"legalFormId":1,"firstname":"Repro","lastname":"C'$N'","active":true,"activationDate":"01 October 2026","submittedOnDate":"01 October 2026",'"$D"'}' | jq -r .resourceId)
SP=$(post /savingsproducts '{"name":"Repro savings '$N'","shortName":"'$N'","currencyCode":"USD","digitsAfterDecimal":2,"inMultiplesOf":0,"nominalAnnualInterestRate":0,"interestCompoundingPeriodType":1,"interestPostingPeriodType":4,"interestCalculationType":1,"interestCalculationDaysInYearType":365,"accountingRule":1,"locale":"en"}' | jq -r .resourceId)
SAVINGS=$(post /savingsaccounts '{"clientId":'$CLIENT',"productId":'$SP',"submittedOnDate":"01 October 2026",'"$D"'}' | jq -r .resourceId)
post "/savingsaccounts/$SAVINGS?command=approve" '{"approvedOnDate":"01 October 2026",'"$D"'}'
post "/savingsaccounts/$SAVINGS?command=activate" '{"activatedOnDate":"01 October 2026",'"$D"'}'

# 2. A product that pays out in up to two tranches, and a loan of 1000 linked to the savings account
LP=$(post /loanproducts '{"name":"Repro tranches '$N'","shortName":"'$N'","currencyCode":"USD","digitsAfterDecimal":2,"inMultiplesOf":0,"principal":1000,"minPrincipal":1,"maxPrincipal":1000000,"numberOfRepayments":4,"repaymentEvery":1,"repaymentFrequencyType":2,"interestRatePerPeriod":0,"interestRateFrequencyType":2,"interestType":0,"interestCalculationPeriodType":0,"amortizationType":1,"inArrearsTolerance":0,"transactionProcessingStrategyCode":"mifos-standard-strategy","accountingRule":1,"daysInMonthType":1,"daysInYearType":1,"isInterestRecalculationEnabled":false,"loanScheduleType":"CUMULATIVE","isEqualAmortization":false,"multiDisburseLoan":true,"maxTrancheCount":2,"outstandingLoanBalance":1000,"disallowExpectedDisbursements":true,'"$D"'}' | jq -r .resourceId)
LOAN=$(post /loans '{"clientId":'$CLIENT',"productId":'$LP',"principal":1000,"loanTermFrequency":4,"loanTermFrequencyType":2,"numberOfRepayments":4,"repaymentEvery":1,"repaymentFrequencyType":2,"interestRatePerPeriod":0,"amortizationType":1,"interestType":0,"interestCalculationPeriodType":0,"transactionProcessingStrategyCode":"mifos-standard-strategy","expectedDisbursementDate":"01 October 2026","submittedOnDate":"01 October 2026","loanType":"individual","linkAccountId":'$SAVINGS',"maxOutstandingLoanBalance":1000,'"$D"'}' | jq -r .resourceId)
post "/loans/$LOAN?command=approve" '{"approvedOnDate":"01 October 2026","approvedLoanAmount":1000,"expectedDisbursementDate":"01 October 2026",'"$D"'}'

# 3. Two tranches paid into the savings account: 600 on 5 October, 400 on 10 October
post "/loans/$LOAN?command=disburseToSavings" '{"actualDisbursementDate":"05 October 2026","transactionAmount":600,'"$D"'}'
post "/loans/$LOAN?command=disburseToSavings" '{"actualDisbursementDate":"10 October 2026","transactionAmount":400,'"$D"'}'
DISB2=$(get "/loans/$LOAN?associations=transactions" | jq '.transactions[] | select(.amount == 400) | .id')
SAVINGS_TX=$(get "/savingsaccounts/$SAVINGS?associations=transactions" | jq '.transactions[] | select(.amount == 400) | .id')
TRANSFER=$(get "/savingsaccounts/$SAVINGS?associations=transactions" | jq '.transactions[] | select(.amount == 400) | .transfer.id')
get "/loans/$LOAN?associations=transactions" | loan_view
get "/savingsaccounts/$SAVINGS?associations=transactions" | savings_view

# 4. Undo the last tranche: 200
post "/loans/$LOAN?command=undolastdisbursal" '{"note":"undo the 400 tranche"}'

# 5. The loan no longer has the 400; the savings account still has it, and the transfer stands
get "/loans/$LOAN?associations=transactions" | loan_view
get "/loans/$LOAN/transactions/$DISB2" | jq -c '{id, type: .type.value, amount, reversedOnDate, transfer: {id: .transfer.id, reversed: .transfer.reversed}}'
get "/savingsaccounts/$SAVINGS?associations=transactions" | savings_view
get "/accounttransfers/$TRANSFER" | jq -c '{id, transferAmount, reversed}'

# 6. Neither side can be undone now
post "/accounttransfers/$TRANSFER?command=undo" '{}' | jq -c '{httpStatusCode, message: .errors[0].developerMessage}'
post "/savingsaccounts/$SAVINGS/transactions/$SAVINGS_TX?command=undo" '{}' | jq -c '{httpStatusCode, message: .errors[0].developerMessage}'
```

### Where in the code

`LoanWritePlatformServiceJpaRepositoryImpl` (fineract-provider), line numbers as in 1.16.1:
- `undoLoanDisbursal` (line 899), the full undo, calls `accountTransfersWritePlatformService.reverseAllTransactions(loanId, PortfolioAccountType.LOAN)` (line 935). That is why the full undo also reverses the savings deposits.
- `undoLastLoanDisbursal` (line 2652) goes through `undoLastDisbursal` (line 3443) and `reverseExistingTransactionsTillLastDisbursal` (line 3502), which reverse the loan's own transactions only. None of the three calls the account transfer service, so the transfer and its savings deposit stay.

On `develop` (de5c64036abe) the same four methods start at lines 855, 2076, 2664 and 2718.

### Possible fix

`AccountTransfersWritePlatformService.reverseTransfersWithFromAccountTransactions(Collection<Long>, PortfolioAccountType)` already reverses the transfers whose loan side is one of the given loan transactions, savings deposit included. Calling it from `undoLastLoanDisbursal` with the ids of the loan transactions that `reverseExistingTransactionsTillLastDisbursal` reverses would do for the last tranche what the full undo does for all of them. We have not built or tested this.

---

Posted to the project: https://issues.apache.org/jira/browse/FINERACT-3320
