# Cosegan community service

As part of Cosegan’s community service initiative, we examined the following open source projects and noticed a few bugs we will share here. We also reported all these through the preferred channels for each project.

At Cosegan, we formally verify critical software (using mathematical proofs). If the proofs pass, then we certify certain functionality, and you don’t have to worry that there are lurking bugs that AI will find later. However, if they don’t pass (like happened here), we know there is likely a bug in the code, even if we don’t yet know what that bug is. In your case, I drilled down to find out why the proofs didn’t pass and came up with this. If you have any questions please contact me at Eric@cosegan.com.

Formal verification is not new, but it’s normally very expensive so it’s only done on critical code like safety of flight or cryptographic libraries. I’m attempting to do it here for these projects as community service, since they are open source and usually free to use.

If you would like Cosegan to analyze or formally verify your code as well, please reach out to Eric@Cosegan.com!

| Project | Version | Findings | Found |
|---|---|---|---|
| [Kill Bill](killbill/README.md) | 0.24.22, with its Stripe plugin 8.0.4 and Braintree plugin 1.0.3 | 9 | 2026-11-12 to 2026-11-16 |
| [Hyperswitch](hyperswitch/README.md) | v1.129.0 | 11, and 1 reported privately | 2026-11-20 |
| [Apache Fineract](fineract/README.md) | 1.16.1 | 2 | 2026-11-20 |
| [Lago](lago/README.md) | v1.57.0 | 2, and 1 reported privately | 2026-11-20 |
| [IJPay](ijpay/README.md) | 3.0.0 | 1 | 2026-11-20 |
