# Apache Fineract 1.16.1

Two findings in the loan and savings code of [Apache Fineract](https://github.com/apache/fineract), release 1.16.1. Found by [Cosegan](https://cosegan.com) on 2026-11-20.

| # | Finding | In | Report | Posted to the project |
|---|---|---|---|---|
| 1 | Undoing the last tranche leaves it in the linked savings account: the client keeps money the loan no longer shows as lent | Loan, Savings | [report](undo-last-tranche-leaves-money-in-savings.md) | https://issues.apache.org/jira/browse/FINERACT-3320 |
| 2 | Undoing the last tranche erases a repayment made the same day: the loan shows the payment as never made | Loan | [report](undo-last-tranche-erases-same-day-repayment.md) | https://issues.apache.org/jira/browse/FINERACT-3321 |
