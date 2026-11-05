# Heimdall: stopping UPI mule-account money inside the bank, while the payment is happening

*Named after Heimdall, the Norse watchman who guards the gods' bridge, needs less sleep than a bird and hears the grass grow.*

**In one line:** when money arrives in an account that behaves like a *mule account*, the receiving bank holds that amount **during the UPI payment**, before it can be passed on, and then follows RBI's draft hold procedure.

**Status:** research prototype for the AIMIT FinTech Innovation Hackathon 2026, Track 1 (Digital Payments Fraud Detection & Prevention). The model benchmark is complete and reproducible (Section 4). The live, transaction-by-transaction engine is the next stage.

---

## 1. The problem

- Fraudsters do not keep stolen money in their own accounts. They pass it through other people's accounts, called **mule accounts**, and move it on quickly.
- Banks usually act **after** a victim complains or the police report an account. By then the money has often already left.
- RBI's draft *Know Your Customer (Amendment) Directions, 2026* (11 Sep 2026; banks to implement by 1 Apr 2027) set out how banks may **temporarily hold** suspected mule transactions of ₹1,000 and above:
  - tell the customer immediately (or by the end of the next day);
  - give the customer 20 days to explain;
  - decide within 10 days of the reply (or 30 days from the hold if there is no reply);
  - release the money, or report to the police through NCRP-CFCFRMS;
  - hold for at most 60 days unless the police or another competent authority instruct otherwise.

  Holding a whole account is meant to be a last resort.
- **The open question:** can a bank decide, while the payment is still in progress, which incoming money is risky enough to hold? NPCI allows 15 seconds for the whole UPI payment.

## 2. What we propose

![Figure 1: what happens during one UPI payment](figures/fig1_payment_flow.png)

- **Where it sits:** inside the **receiving bank**, next to the fraud checks the bank already runs. Nothing changes at NPCI.
- **What it reads:** each account's **risk record**, a few running numbers such as how many different people it deals with, how many of those relationships go both ways, and how much of the money it receives it sends on. Every transaction updates these numbers.
- **What it decides:** a model, trained beforehand on past mule accounts, turns the numbers into a **risk score**. The bank then takes one of three actions:
  - **Low risk:** credit normally.
  - **Medium risk:** credit, watch the account, and ask the customer to confirm large outgoing transfers.
  - **High risk, and the amount is ₹1,000 or more:** credit but **hold this amount**. RBI's draft procedure starts, and a person reviews every hold.
- **What "real time" means here:** the *action* happens during the payment. The model itself is trained in advance and refreshed regularly.

## 3. How we tested it

![Figure 2: how the methods were compared](figures/fig2_benchmark_flow.png)

**The question:** *on 1 January 2025, using only transactions before that date, which accounts will later be flagged as mules?*

**The rules:**
- **No future data:** only transactions before 1 Jan 2025 are used.
- **Full histories:** each account's transactions are joined from all files. About 7% of accounts, and about half of the mules, are split across files.
- **Fair comparison:**
  - every method gets the same 17 features and default settings;
  - the test is repeated over 5 random 70/30 account splits;
  - test accounts are never seen during training.

**What was tested:** 92,990 accounts, of which 1,880 (2.0%) are mules that the bank had not yet flagged on 1 Jan 2025. A further 651 mules that were already flagged before that date were removed.

## 4. Results (average ± spread over 5 splits)

| Method | ROC-AUC | PR-AUC | Mules caught when 1 in 100 genuine accounts is wrongly flagged | … when 1 in 1,000 is | Time to score one account* |
|---|---|---|---|---|---|
| Simple rule (no learning) | 0.82 ± 0.01 | 0.09 ± 0.00 | 3% | 0.2% | – |
| Logistic regression | 0.88 ± 0.01 | 0.18 ± 0.01 | 16% | 3% | 2.1 ms |
| Random forest | 0.91 ± 0.01 | 0.56 ± 0.01 | 61% | 19% | 111 ms |
| **Gradient boosting (LightGBM)** | **0.91 ± 0.01** | **0.56 ± 0.01** | **62%** | **18%** | **1.6 ms** |
| *Check:* LightGBM without the two-way feature | 0.90 ± 0.01 | 0.53 ± 0.01 | 60% | 16% | – |

\*Median time to score one account, measured in the Kaggle notebook on Kaggle's standard CPU machine (no GPU).

**How to read the scores**
- **ROC-AUC:** take one mule and one genuine account at random. How often does the mule get the higher score? 0.5 is a coin toss; 1.0 is perfect.
- **PR-AUC:** of the accounts a method flags, how many are really mules? A coin toss scores about 0.02 here.
- **Mules caught at a fixed false-alarm level:** how many mules are found if the bank accepts wrongly flagging 1 in 100 (or 1 in 1,000) genuine customers.

**The same result in people** (one test split, LightGBM; 27,897 test accounts, of which 564 are mules):

| Setting | Genuine customers wrongly flagged | Mules caught | Alerts that are real mules |
|---|---|---|---|
| 1 in 100 genuine accounts flagged | 274 | 367 (65%) | 57% |
| 1 in 1,000 genuine accounts flagged | 28 | 87 (15%) | 76% |

**What we learned**
1. **Random forest and LightGBM detect equally well.** The difference is smaller than the spread between splits. We use LightGBM because it scores one account in 1.6 ms (random forest: 111 ms with default settings).
2. **The simple rule is close to useless** at realistic false-alarm levels (3% of mules caught). A learned model is needed.
3. **The result does not rest on one pattern.** In this synthetic data, 91% of genuine accounts have *every* contact two-way, which is unlikely in a real bank. Removing that feature lowers PR-AUC only from 0.56 to 0.53.
4. **There is a real trade-off.** Catching more mules means wrongly flagging more genuine customers. The bank should set the hold level by how many cases its staff can review each day, and use "watch/confirm" rather than "hold" for medium scores.
5. **Fairness gap:** at the same setting, genuine **rural** customers were wrongly flagged 1.31% of the time against 0.96% for urban customers, about 1.4× more often. Rural flags should therefore go to a person before any hold is placed.

**Data checks (possible traps in the data)**
- **Impossible dates:** 323 accounts have an opening date *after* 1 Jan 2025 even though they transacted before it, and 4,318 were "opened" after their first transaction. This cannot happen in a real bank, yet these accounts are mules far more often (33% and 10%, against 2% overall), so it may be one of the dataset's planted red herrings. Removing the account-age feature changes LightGBM's PR-AUC only from 0.56 to 0.55, within the spread, so the model does not depend on it. *(Checked separately; not yet a cell in the notebook.)*
- **Dormant accounts:** 26% of mules had no transaction in the 30 days before the cutoff, against 4% of genuine accounts. *Caution:* the dataset's organisers deliberately planted genuine "dormant then reactivated" accounts as a trap (one of the challenge's red-herring groups, as reported by the [DhanRakshak](https://github.com/shivangiS04/DhanRakshak) submission), so this signal may not hold on harder test sets.

## 5. Speed and scale

- **Speed:** scoring one account takes about 1.6 ms on Kaggle's standard CPU (measured above). Updating an account's risk record is a handful of additions per transaction. The full transaction-by-transaction engine is the next stage, and its end-to-end timing is **not yet measured**.
- **Scale:** UPI processed 24.07 billion transactions in September 2026, about 9,300 per second on average. Each bank checks only its own customers, so this load is naturally divided among banks.
- **Open question:** keeping a contact list for crores of accounts needs a fast data store (for example Redis), and its memory cost must be measured.

## 6. What others already do

- **Mastercard Consumer Fraud Risk (UK, 2023):** real-time scoring of bank-to-bank payments using links to mule accounts, used by nine UK banks. This confirms the approach is mainstream.
- **Brazil, Pix (Nov 2021):** the receiving institution may block funds received by an individual for up to **72 hours** on suspected fraud, and must tell the customer. This is the closest precedent for "hold the amount when it lands".
- **United Kingdom (Oct 2024):** the sending bank may delay an outgoing payment for up to **4 business days** on reasonable grounds of fraud, telling the customer. This is the precedent for "confirm before sending".
- **Open-source projects:** most "real-time UPI fraud" repositories use self-generated data and report untested latency. We followed the stronger practices: past-only data, a simple-rule baseline, results at fixed false-alarm levels, repeated splits, and stated limitations.

## 7. Limitations (please read)

- **The data is synthetic** (RBIH NFPC). Patterns here, for example almost every genuine account having two-way relationships, may not hold in a real bank.
- **Labels are the bank's own flags.** Most mules in this dataset were flagged in a single sweep (April–June 2025), and any undetected mules are counted as genuine.
- **One cutoff date and no tuning, by design.** Different dates or tuned methods could give somewhat different numbers.
- **No live bank traffic.** Real-world validation would need a partner bank, and a production version would need the bank's own data, because the dataset licence is non-commercial.

## 8. Reproduce the results

1. Open the notebook on Kaggle: **https://www.kaggle.com/code/gokuljayachandran/heimdall-benchmark**, or upload `notebooks/heimdall_benchmark.ipynb` with **File → Import notebook**.
2. Attach the dataset `abhyudayrbih/rbih-nfpc-phase-2` and set Accelerator to *None*.
3. Click **Run All**. The full run takes about 20–30 minutes; for a quick trial, set `N_FILES = 20`.

```
heimdall-upi/
├── README.md
├── LICENSE                                MIT licence for the code
├── notebooks/heimdall_benchmark.ipynb   all steps of Figure 2, with plain explanations
├── figures/                               Figure 1 and Figure 2
└── results/                               benchmark_summary.csv and benchmark_results_by_split.csv (the notebook's outputs)
```

## 9. Data, licence, originality

- **Code:** MIT licence (see [`LICENSE`](LICENSE)).
- **Data:** RBIH × IIT Delhi National Fraud Prevention Challenge Phase-2 dataset (Kaggle), CC BY-NC-SA 4.0 (non-commercial). The data is not included in this repository and keeps its own licence. No confidential or real customer data is used anywhere in this project.
- **Originality:** this is our own work, submitted by **Gokul Jayachandran** (team **MARK XLVI**, gokuljai2006@gmail.com), Sri Sai Ram Engineering College, Chennai.

## References

1. RBI, Draft *Reserve Bank of India (Know Your Customer) Amendment Directions, 2026*, SOP for temporary debit holds, 11 Sep 2026.
2. NPCI, UPI-OC No. 214 (2025-26), "Reduction in response time of UPI transaction", 26 Apr 2025 (Request/Response Pay limit 15 s from 16 Jun 2025).
3. NPCI UPI statistics, September 2026: 24.07 billion transactions.
4. Mastercard, "Mastercard leverages its AI capabilities to fight real-time payment scams", 6 Jul 2023.
5. Banco Central do Brasil: preventive 72-hour block of Pix funds on suspected fraud, effective 16 Nov 2021 (as reported by InfoMoney, 29 Sep 2021).
6. UK Payment Services (Amendment) Regulations 2024: delay of outbound payments up to 4 business days (in force Oct 2024).
7. RBIH & IIT Delhi, NFPC Phase-2 dataset, Kaggle `abhyudayrbih/rbih-nfpc-phase-2`.
