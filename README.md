# Credit Risk Modelling in Python: PD, LGD, EAD and Expected Loss

This project builds, from scratch and step by step, the three models a bank uses to measure **credit risk**:

- **PD**: will the borrower default?
- **LGD**: if they default, how much of the money is lost?
- **EAD**: how much is still owed at the moment of default?

It then combines them into the **Expected Loss** of a portfolio of about 466,000 real loans.

Everything is in one notebook, [`Credit_Risk_Modelling.ipynb`](Credit_Risk_Modelling.ipynb), written for beginners. Every important step explains **what** we do, **why**, **how to read the result**, **what goes wrong if we skip it**, and the **common beginner mistake** at that step.

---

## Contents
1. [The idea in plain English](#1-the-idea-in-plain-english)
2. [Results at a glance](#2-results-at-a-glance)
3. [Repository structure](#3-repository-structure)
4. [The dataset](#4-the-dataset)
5. [How to run](#5-how-to-run)
6. [What the notebook does, step by step](#6-what-the-notebook-does-step-by-step)
7. [Variable dictionary](#7-variable-dictionary)
8. [Concepts explained (with common beginner mistakes)](#8-concepts-explained-with-common-beginner-mistakes)
9. [Code review: what was fixed and changed](#9-code-review-what-was-fixed-and-changed)
10. [Limitations and next steps](#10-limitations-and-next-steps)

---

## 1. The idea in plain English

A bank cannot know in advance *which* borrower will stop paying, but it can estimate the **average** loss over many loans, the same way an insurer estimates how many cars will crash without knowing which ones.

> **Expected Loss (EL) = PD × LGD × EAD**

| Part | Question | Example |
|---|---|---|
| **PD**: Probability of Default | How likely is it that the borrower stops paying (within a stated period)? | 0.08 (8 out of 100 such borrowers default) |
| **LGD**: Loss Given Default | If they stop paying, what share of the money still owed is lost? | 0.90 (only 10% is recovered) |
| **EAD**: Exposure at Default | If they stop paying, how much is still owed at that moment? | $7,500 |
| **EL**: Expected Loss | How much do we expect to lose on this loan, on average? | 0.08 × 0.90 × $7,500 = **$540** |

Banks use these numbers to **price** loans (riskier borrowers pay higher interest), to **approve or reject** applications, to set money aside for future losses (**provisions**, IFRS 9 / CECL), and to calculate how much **capital** regulators require (Basel II / III).

| Model | Built on | Target (what we predict) | Technique |
|---|---|---|---|
| PD | all loans | good (1) or bad (0) | logistic regression, turned into a 300-850 scorecard |
| LGD | charged-off loans only | recovery rate = net recoveries ÷ amount owed at default | two stages: logistic regression + linear regression |
| EAD | charged-off loans only | CCF = amount owed at default ÷ amount lent | linear regression |
| EL | all loans | PD × LGD × EAD | combine the three models |

---

## 2. Results at a glance

All numbers are measured on **test data** the models never saw while learning (except the portfolio Expected Loss, which covers every loan).

| Model | Measure | Result | In plain English |
|---|---|---|---|
| PD | AUC | **0.684** | a random bad loan gets a higher PD than a random good loan 68% of the time (0.5 = coin flip) |
| PD | Gini | **0.368** | 2 × AUC − 1 |
| PD | KS | **0.269** | largest gap between the good and bad loans' PD distributions |
| PD | Calibration | predicted PD ≈ actual bad rate in all 10 risk groups | a PD of 10% really means about 10% bad loans |
| PD | Overfitting check | training AUC 0.680 vs test AUC 0.684 | no overfitting |
| Scorecard | Example cut-off 600 | approves 74% of loans; their bad rate falls from 10.9% to 7.8% | the score separates risk well |
| LGD stage 1 | AUC | **0.656** | "will anything be recovered?" is predicted better than chance |
| LGD stage 2 | R² | 0.028 | *how much* is recovered is very hard to predict from application data (normal for LGD) |
| LGD combined | average recovery rate | predicted 7.3% vs actual 7.5% | right on average |
| EAD | R² / correlation | 0.20 / 0.46 | moderate; average CCF predicted 0.737 vs actual 0.734 |
| **Expected Loss** | whole portfolio | **$509 million on $6.66 billion lent = 7.6%** | from ≈ 2.1% (grade A) to ≈ 20.5% (grade G) |

**Why is the PD AUC "only" 0.68?** The data contains only loans that Lending Club had already **approved**, so the riskiest applicants were filtered out before they reached the data. Application scorecards on such data typically land around 0.65 to 0.75.

---

## 3. Repository structure

```
Credit-Risk-Modeling/
├── Credit_Risk_Modelling.ipynb   # the complete project, step by step, with notes and saved outputs
├── DataDictionary.xlsx           # Lending Club's official definition of every column
├── requirements.txt              # Python libraries needed to run the notebook
├── .gitignore                    # keeps the large data file out of git
└── README.md                     # this file
```

---

## 4. The dataset

- **Source:** [Kaggle: loan data 2007-2014](https://www.kaggle.com/datasets/devanshi23/loan-data-2007-2014). The loans come from Lending Club, a US **peer-to-peer lender**: a website that connects borrowers with investors who lend them the money.
- **Size:** 466,285 loans × 75 columns, 240 MB (`loan_data_2007_2014.csv`).
- **One row = one loan:** information known when the loan was given (amount, interest rate, grade, income, credit history, ...) plus what happened afterwards (`loan_status`, payments, recoveries), as of early 2016.
- **Not stored in this repository:** GitHub rejects files larger than 100 MB, and `.gitignore` stops the CSV from being committed by accident.

---

## 5. How to run

### Option A: Google Colab (nothing to install)
1. Open the notebook in Colab: <https://colab.research.google.com/github/imdsarfraz/Credit-Risk-Modeling/blob/main/Credit_Risk_Modelling.ipynb>
2. Get the data into Colab, in one of two ways:
   - download `loan_data_2007_2014.csv` from Kaggle and upload it with the folder icon on the left, **or**
   - run this in a new cell (Kaggle's public download link at the time of writing):
     ```
     !curl -L -o archive.zip https://www.kaggle.com/api/v1/datasets/download/devanshi23/loan-data-2007-2014
     !unzip -o archive.zip && mv loan_data_2007_2014/loan_data_2007_2014.csv .
     ```
3. **Runtime → Run all.** The whole notebook runs in a few minutes and needs about 4 GB of RAM.

### Option B: on your own computer
```bash
git clone https://github.com/imdsarfraz/Credit-Risk-Modeling.git
cd Credit-Risk-Modeling
python -m venv .venv
.venv\Scripts\activate          # Windows   (Mac/Linux: source .venv/bin/activate)
pip install -r requirements.txt
# put loan_data_2007_2014.csv in this folder, then:
jupyter notebook Credit_Risk_Modelling.ipynb
```

Tested with Python 3.11 and both **pandas 3.1** (numpy 2.4, statsmodels 0.15, scikit-learn 1.9) and **pandas 2.2** (numpy 1.26, statsmodels 0.14, scikit-learn 1.5); both give identical results.

---

## 6. What the notebook does, step by step

| Part | Step | Key idea |
|---|---|---|
| 0 | The big picture | EL = PD × LGD × EAD; why accuracy is misleading for credit risk |
| 1 | Setup and loading | `low_memory=False`; keep an untouched backup copy |
| 2 | First look + variable dictionary | which columns are inputs, which are the answer, which would leak the answer |
| 3 | General data preparation | text → numbers, dates → months (with the "year 2068" bug fixed), dummy variables, missing values |
| 4 | Good / bad definition | 1 = good, 0 = bad (charged off, default, 31+ days late); bad rate 10.9%; what "PD" means for this data |
| 5 | Train / test split | 80% to learn, 20% sealed "exam paper"; `random_state` for reproducibility |
| 6 | WoE and IV, fine and coarse classing | 24 variables analysed: risk pattern, merged categories, and two "loan age" traps found and avoided |
| 7 | Final variables | every loan in exactly one category (automatic check), IV table, reference categories, keep IV ≥ 0.02 (11 variables) |
| 8 | Logistic regression | coefficients, odds ratios, p-values, interpretation, multicollinearity |
| 9 | Validation | confusion matrix and the accuracy trap, ROC/AUC, Gini, KS, calibration, overfitting check |
| 10 | Scorecard | points per category, scores 300-850, bad rate by score band, cut-off trade-off, score → PD table |
| 11 | LGD model | recovery rate on the amount owed; why two stages (44% recover nothing); expected recovery = P(recovery) × amount |
| 12 | EAD model | CCF = share still owed at default; EAD = CCF × amount |
| 13 | Expected Loss | PD × LGD × EAD for every loan; total portfolio loss; loss by grade |
| 14 | Summary | results, honest limitations, next steps |

---

## 7. Variable dictionary

### 7.1 Original columns used

| Column | Meaning in plain English | Example | Used in |
|---|---|---|---|
| `id` | unique number of the loan | 1077501 | identification only (never a model input) |
| `funded_amnt` | amount actually lent ($) | 5000 | EAD, EL, input for LGD/EAD |
| `term` | number of monthly payments: 36 or 60 | ' 36 months' | PD, LGD, EAD |
| `int_rate` | yearly interest rate (%) | 10.65 | PD, LGD, EAD |
| `installment` | monthly payment ($) | 162.87 | analysed, not used (repeats amount + rate + term) |
| `grade` | Lending Club's risk grade: A = safest ... G = riskiest | 'B' | PD, LGD, EAD |
| `emp_length` | years in the current job | '10+ years' | PD analysis, LGD, EAD |
| `home_ownership` | RENT, OWN, MORTGAGE (paying off a home loan), OTHER, NONE, ANY | 'RENT' | PD, LGD, EAD |
| `annual_inc` | yearly income stated by the borrower ($) | 24000 | PD, LGD, EAD |
| `verification_status` | was the income checked? Verified / Source Verified / Not Verified | 'Verified' | PD, LGD, EAD |
| `issue_d` | month the loan was given | 'Dec-11' | time calculations |
| `loan_status` | what happened to the loan: Fully Paid, Charged Off, Current, ... | 'Fully Paid' | **target** of the PD model; selects charged-off loans for LGD/EAD |
| `purpose` | reason for the loan | 'credit_card' | PD, LGD, EAD |
| `addr_state` | US state of the borrower | 'AZ' | PD analysis |
| `dti` | debt-to-income: monthly debt payments (without mortgage and this loan) ÷ monthly income (%) | 27.65 | PD, LGD, EAD |
| `delinq_2yrs` | times 30+ days late on any credit in the last 2 years | 0 | PD analysis, LGD, EAD |
| `earliest_cr_line` | month of the borrower's first credit account | 'Jan-85' | length of credit history |
| `inq_last_6mths` | credit applications (enquiries) in the last 6 months | 1 | PD, LGD, EAD |
| `mths_since_last_delinq` | months since the last late payment (empty = never late) | 35 | PD analysis, LGD, EAD |
| `mths_since_last_record` | months since the last negative public record (empty = none) | 113 | PD analysis, LGD, EAD |
| `open_acc` | number of open credit accounts | 3 | PD analysis, LGD, EAD |
| `pub_rec` | number of negative public records (bankruptcies, tax liens, ...) | 0 | PD analysis, LGD, EAD |
| `total_acc` | total number of credit accounts in the credit file (open and closed) | 9 | PD analysis, LGD, EAD |
| `initial_list_status` | how the loan was offered to investors: f = in fractions, w = whole | 'f' | PD, LGD, EAD |
| `acc_now_delinq` | accounts on which the borrower is late right now | 0 | PD analysis, LGD, EAD |
| `total_rev_hi_lim` | total credit limit on revolving accounts such as credit cards ($) | 9000 | analysed, **not used** (empty for all 2007-2011 loans, see Section 8) |
| `recoveries` | money recovered **after** the loan was written off ($, before fees) | 0 | builds the LGD target only |
| `collection_recovery_fee` | fee kept by the collection agency out of those recoveries ($) | 0 | builds the LGD target only |
| `total_rec_prncp` | part of the loan amount (principal) repaid so far ($) | 5000 | builds the LGD and EAD targets only |

"PD analysis" = studied in Part 6, but dropped from the final PD model because its Information Value was below 0.02.

### 7.2 Columns we create

| Column | Meaning | Created in |
|---|---|---|
| `emp_length_int` | years in the current job as a number, 0-10 (empty → 0) | Part 3.1 |
| `term_int` | 36 or 60 as a number | Part 3.2 |
| `issue_d_date`, `earliest_cr_line_date` | real dates (two-digit-year bug fixed) | Part 3.3 |
| `mths_since_earliest_cr_line` | months of credit history **at the date the loan was given** | Part 3.3 |
| `mths_since_issue_d` | loan age in months up to 1 Dec 2017; analysed but **not** used (a trap, see Part 6) | Part 3.4 |
| `grade:A`, `purpose:car`, ... | dummy variables: 1 if the loan is in that category, else 0 | Parts 3.5 and 6 |
| `good_bad` | target of the PD model: 1 = good, 0 = bad | Part 4 |
| `P_good`, `PD` | model's probability of being good; PD = 1 − P_good | Part 9 |
| `score` | credit score, 300 (riskiest) to 850 (safest) | Part 10 |
| `never_delinquent`, `never_public_record` | 1 if the "months since" value was empty, i.e. it never happened | Part 11 |
| `mths_since_last_delinq_filled`, `mths_since_last_record_filled` | copies with empty values replaced by 0 (the flags above carry the "never") | Part 11 |
| `amount_owed_at_default` | funded amount − principal repaid | Part 11 |
| `recovery_rate` | (recoveries − collection fee) ÷ amount owed at default, kept between 0 and 1 | Part 11 |
| `recovery_rate_0_1` | 1 if anything was recovered, else 0 (target of LGD stage 1) | Part 11 |
| `CCF` | amount owed at default ÷ funded amount | Part 12 |
| `LGD`, `EAD`, `EL` | predicted loss share, exposure ($) and expected loss ($) of every loan | Part 13 |

### 7.3 Main Python objects in the code

| Name | What it holds |
|---|---|
| `backup_data` | the original data exactly as read from the file (never changed) |
| `loan_data` | the working copy that we clean |
| `loan_data_inputs_train` / `_test` | inputs of the training (80%) and test (20%) loans |
| `loan_data_targets_train` / `_test` | the good/bad answers of the same loans |
| `woe_table()`, `plot_by_woe()` | helper functions: WoE/IV table of one variable, and its chart |
| `woe_df` | the WoE table of the variable currently being analysed |
| `categories_by_variable` | dictionary: variable → list of its final categories |
| `reference_categories` | dictionary: variable → its riskiest category (left out of the model) |
| `iv_summary` | Information Value of every candidate variable |
| `selected_variables`, `features` | the variables kept (IV ≥ 0.02) and their dummy columns |
| `pd_model` | the fitted logistic regression (PD model) |
| `pd_results_test` | test loans: true answer, P(good), PD, score |
| `scorecard` | points for every category |
| `loan_data_defaults` | charged-off loans only (for LGD and EAD) |
| `lgd_stage_1`, `lgd_stage_2`, `ead_model` | the fitted LGD and EAD models |

### 7.4 Columns deliberately NOT used as inputs
- **Identifiers** (`id`, `member_id`, `url`): say nothing about risk; a model may still find random patterns in them.
- **Free text** (`desc`, `title`, `emp_title`): would need text mining.
- **Information that only exists after the loan was given** (`total_pymnt`, `out_prncp`, `last_pymnt_d`, `recoveries`, ...): using them to predict default is **data leakage**.
- **`sub_grade`**: repeats `grade` in finer detail.
- **Loan age and `total_rev_hi_lim`**: both secretly measure *how old* the loan is (see "loan-age trap" below).
- **17 completely empty columns**.

---

## 8. Concepts explained (with common beginner mistakes)

### Data preparation

**Data leakage**
- *Meaning:* the model is given information that would not be available at the moment of the real decision, often the answer itself in disguise.
- *Why it matters:* the model looks brilliant on our data and fails on real customers.
- *Beginner mistake:* using after-the-fact columns (payments, recoveries), creating dummy variables from the target (`loan_status`), or analysing the test data before modelling.

**The loan-age trap (a sneaky kind of leakage)**
- *Meaning:* `Current` loans count as good, so young loans look safe only because they have not had time to default yet. Anything that measures the loan's age (months since issue, a field that was only collected from 2012 on, a `dti` limit that was only allowed in 2014) "predicts" default for the wrong reason.
- *In this project:* loan age itself, `total_rev_hi_lim` (empty for every 2007-2011 loan) and `dti > 35` (only 2014 loans) all carried this trap; all three are handled in Part 6.
- *Beginner mistake:* keeping a variable because it raises the IV or AUC, without asking "would I know this at decision time, and does it describe the borrower or just the calendar?"

**Dummy variable (one-hot encoding)**
- *Meaning:* one 0/1 column per category (`grade:B = 1` means "grade B").
- *Why:* models need numbers, and coding categories as 1, 2, 3 would force an order and equal steps.
- *If skipped:* text cannot enter a regression.
- *Beginner mistake:* putting all dummies of a variable plus an intercept into the model (the "dummy variable trap", see below).

**Missing values and imputation**
- *Meaning:* empty cells (`NaN`). We either fill them (impute) or treat "missing" as information.
- *Why:* regressions cannot handle empty cells.
- *Rule:* first ask **why** a value is missing. "Never late" is information (own category); "not collected before 2012" is a hidden date (don't use it).
- *Beginner mistakes:* `df['col'].fillna(0, inplace=True)`, which does **nothing** in pandas 3 (write `df['col'] = df['col'].fillna(0)`); filling with 0 when 0 has a real meaning ("0 months since last late payment" ≠ "never late").

**Train / test split**
- *Meaning:* 80% of the data to learn from, 20% kept aside to test.
- *Why:* only unseen data tells us how the model works on new customers.
- *If skipped:* overfitting goes unnoticed.
- *Beginner mistakes:* no `random_state` (results change every run); computing WoE, bins or averages on all the data before splitting.

**Class imbalance**
- *Meaning:* far more good loans (89%) than bad loans (11%).
- *Why it matters:* "everyone is good" is already 89% accurate, so accuracy is meaningless here.
- *Beginner mistake:* reporting accuracy for imbalanced problems.

### PD model

**Default / good-bad definition and the PD time period**
- *Meaning:* bad = Charged Off, Default, Late (31-120 days), "Does not meet the credit policy. Status:Charged Off"; everything else = good.
- *Why 1 = good:* scorecard tradition (higher score = safer). The model therefore predicts P(good), and **PD = 1 − P(good)**.
- *Time period:* a PD always needs one (12 months for capital rules, the loan's life for provisions). Here it is "bad by the time the data was collected", a mix of both.
- *Beginner mistakes:* calling P(good) the PD; quoting a PD without its time period.

**Weight of Evidence (WoE)**
- *Meaning:* ln(share of all good loans in a category ÷ share of all bad loans in it). Positive = safer than average, negative = riskier, ±∞ = category too small.
- *Why:* shows the risk pattern of every category and which neighbours can be merged.
- *If skipped:* categories are chosen blindly; tiny categories produce extreme, unstable results.
- *Beginner mistakes:* computing it on all data; ignoring `inf` values; grouping categories that merely "sound similar" instead of neighbours with similar WoE.

**Information Value (IV)**
- *Meaning:* one number per variable: how well its categories separate good from bad. < 0.02 useless, 0.02-0.1 weak, 0.1-0.3 medium, 0.3-0.5 strong, > 0.5 suspicious.
- *Why:* an objective way to choose variables (p-values are almost always "significant" with 373,000 loans).
- *Beginner mistake:* celebrating IV > 0.5 instead of checking for leakage.

**Fine classing and coarse classing**
- *Meaning:* fine = cut a number into many small intervals just to *look* at the WoE; coarse = merge them into a few final categories.
- *Why categories instead of raw numbers:* risk is not always a straight line, outliers are contained, and the scorecard gives points per category.
- *Beginner mistakes:* equal-width bins on data with huge outliers (income: 94% of loans in one bin); gaps or overlaps between ranges (`<` vs `<=`). The notebook checks automatically that every loan is in exactly one category.

**Reference category and the dummy variable trap**
- *Meaning:* for each variable one category (here the riskiest) is left out of the model and becomes the baseline.
- *Why:* with all dummies plus the intercept, one column is an exact copy of others combined, and the model cannot be estimated (singular matrix).
- *Beginner mistake:* forgetting the reference category in the scorecard (it gets 0 points).

**Logistic regression**
- *Meaning:* adds up coefficients into a score `z`, then squeezes it into a probability with `1 / (1 + e^(-z))`.
- *Why:* transparent, explainable, converts directly into a scorecard; preferred by banks and regulators.
- *Beginner mistakes:* reading a coefficient as a probability; forgetting the intercept (`sm.Logit` and `sm.OLS` need `sm.add_constant`).

**Odds and odds ratio**
- *Meaning:* odds = P(good) ÷ P(bad); odds ratio = e^coefficient = how many times the odds change compared with the reference category.
- *Example:* `int_rate:<=9.548` has an odds ratio of about 4.2, so the odds of being good are about 4 times those of a loan above 20.281%.

**p-value / statistical significance**
- *Meaning:* if there were really no effect, how often would an effect at least this big appear by chance? Below 0.05 = "significant". It is **not** the probability that the effect is real.
- *Rule used:* keep a variable if at least one of its categories is significant, and keep or drop whole variables, never single categories. A stricter alternative is one likelihood-ratio test per variable.
- *Beginner mistake:* reading p > 0.05 as proof of "no effect".

**Multicollinearity**
- *Meaning:* two inputs carry almost the same information (here `grade` and `int_rate`).
- *Effect:* predictions are fine, but the individual coefficients become unstable and hard to interpret.
- *Beginner mistake:* interpreting each coefficient on its own in that situation.

**Selection bias**
- *Meaning:* the data only contains loans that were approved, so rejected (riskier) applicants are missing.
- *Effect:* some risk signals look weak (for example past late payments): approved borrowers with late payments had to be strong in other ways.

### Validation

**Confusion matrix and threshold**
- *Meaning:* to turn a probability into a decision you need a threshold; the confusion matrix counts right and wrong decisions.
- *Lesson from the data:* at threshold 0.5 the model calls almost everyone good (89% accuracy, almost no bad loans caught); at 0.9 it catches 72% of the bad loans but accuracy drops to 56%.
- *Beginner mistake:* using accuracy and the default 0.5 threshold.

**ROC curve and AUC**
- *Meaning:* the ROC curve shows, for every threshold, the share of bad loans caught against the share of good loans wrongly flagged. AUC = the chance that a random bad loan gets a higher PD than a random good loan.
- *Scale:* 0.5 = random, 0.7-0.8 = good for application scorecards, > 0.9 = check for leakage, < 0.5 = labels probably swapped.

**Gini coefficient**
- *Meaning:* 2 × AUC − 1 (0 = random, 1 = perfect). The banking industry's favourite way to report AUC.

**Kolmogorov-Smirnov (KS)**
- *Meaning:* the largest gap between the cumulative share of good loans and of bad loans when sorting by PD. 0 = no separation, 1 = perfect.

**Calibration**
- *Meaning:* does a predicted PD of 10% really mean 10% bad loans? Compare the average predicted PD with the actual bad rate in groups, **on test data**.
- *Why:* AUC only checks the *ranking*; Expected Loss needs the *level* to be right too.
- *Beginner mistake:* checking calibration on training data, where a logistic regression matches the bad rate of every category it contains automatically.

**Overfitting**
- *Meaning:* the model memorises the training data instead of learning general patterns.
- *Check:* training AUC (0.680) vs test AUC (0.684), so there is no overfitting.

### Scorecard

**Scorecard / credit score**
- *Meaning:* coefficients rescaled into whole points per category so that scores run from 300 (riskiest possible borrower) to 850 (safest). Score = intercept points + one category's points per variable.
- *Why:* anyone can read and check a points table; every score maps back to exactly one PD.
- *Beginner mistakes:* forgetting the reference categories (0 points); leaving the intercept out of the min/max sums; treating the score like a FICO score (only the 300-850 range is borrowed).

**Cut-off and approval rate**
- *Meaning:* approve applications with a score at or above the cut-off.
- *Trade-off:* a higher cut-off gives fewer bad loans but fewer customers. Choosing it is a business decision, not a statistical one.

### LGD, EAD and Expected Loss

**Recovery rate and LGD**
- *Meaning:* recovery rate = (money recovered after the charge-off − collection fee) ÷ amount owed at default; LGD = 1 − recovery rate.
- *Why the amount owed:* EL multiplies LGD by EAD, so LGD must be a share of that same amount.
- *Data:* only charged-off loans (only they have recoveries).
- *Beginner mistakes:* training on all loans; dividing by the amount lent and then multiplying by EAD (recoveries shrink twice); using gross recoveries; not capping rates above 1.

**Two-stage LGD model**
- *Meaning:* 44% of charged-off loans recover nothing. Stage 1 (logistic regression) predicts *whether* anything is recovered; stage 2 (linear regression, only on loans with recoveries) predicts *how much*. Expected recovery = P(any recovery) × predicted recovery.
- *If skipped:* one straight line cannot fit a target where almost half the values are exactly 0.

**Linear regression and R²**
- *Meaning:* predicts a number as a weighted sum of inputs (OLS = least squares). R² = share of the variation explained (0 = nothing, 1 = everything).
- *Beginner mistake:* trusting predictions outside the possible range; we clip LGD and CCF to 0-1.

**CCF and EAD**
- *Meaning:* CCF = share of the original amount still owed at default; EAD = CCF × amount lent. (In Basel rules "CCF" is narrower: the share of an *unused credit line* drawn before default; for fully paid-out loans the simpler meaning is used.)
- *If skipped (EAD = full amount):* exposure and losses are overstated, because borrowers repay about a quarter of the principal before defaulting.
- *Beginner mistake:* using `out_prncp` as EAD (it is 0 after a charge-off).

**Expected Loss**
- *Meaning:* PD × LGD × EAD in dollars: the **average** loss to expect, like an insurance premium.
- *Beginner mistakes:* using P(good) instead of PD; mixing percentages (90) with shares (0.9); thinking EL is the worst case (bad years bring losses far above EL, which is what bank capital is for).

---

## 9. Code review: what was fixed and changed

The first version of the notebook stopped after preparing the training data for the PD model. The review fixed the issues below and completed the project (PD model, validation, scorecard, LGD, EAD, Expected Loss, README). An independent second review of the finished notebook found more issues, also fixed (rows 17-21).

| # | Issue in the first version | Why it matters | Fix |
|---|---|---|---|
| 1 | `loan_data['col'].fillna(x, inplace=True)` | works with a warning in pandas 2, but **does nothing** in pandas 3 (chained assignment), so on a current pandas the missing values silently stay empty | `loan_data['col'] = loan_data['col'].fillna(x)` |
| 2 | dummy variables created from `loan_status` (the target) | the answer inside the inputs = data leakage trap | removed |
| 3 | categories built with `isin(range(85, int(max)))` | `range` stops *before* the maximum, so the loans with the largest value fell into **no** category; on test data the maximum differs again | plain comparisons (`> 84`), plus an automatic check that every loan is in exactly one category |
| 4 | test data to be prepared by "change the cell to the test set and re-run everything" | error-prone, and the saved notebook only prepared the training data, so no model could be validated | one loop applies the same rules to training and test data; rules come from training data only |
| 5 | credit history measured up to Dec 2017 instead of the loan date | mixes the loan's age into a borrower variable | measured at the issue date; new categories from a new WoE analysis |
| 6 | loan age (`mths_since_issue_d`) used as a predictor | young loans "look" safe only because they had no time to default; for a new applicant it's always 0 | analysed and explained, but excluded from the model |
| 7 | `'(\d+)'` regex pattern | `SyntaxWarning: invalid escape sequence` | raw string `r'(\d+)'` |
| 8 | `DtypeWarning: Columns (20) have mixed types` | type guessing chunk by chunk | `low_memory=False` |
| 9 | `mths_since_issue_d` first held a date, then a number | confusing variable meaning | separate `issue_d_date` and `mths_since_issue_d` |
| 10 | adding columns to filtered copies (`df_inputs_prepr_temp`) | `SettingWithCopyWarning`, risk of changes going nowhere | the WoE function takes the filtered column directly; no temporary copies |
| 11 | `groupby` on intervals without `observed` | `FutureWarning`, and thousands of empty rows (2,000 bins for `total_rev_hi_lim`) | `observed=True`; zoom into the dense range with 50 bins |
| 12 | two almost identical WoE functions; typo `discrete_variabe_name` | duplicated code | one `woe_table(variable, target, sort_by_woe)` |
| 13 | `train_test_split` run twice (first without `random_state`) | unreproducible first split | one reproducible split |
| 14 | comments that contradicted the code ("10 categories of 15k", "more than 150K") | misleading | comments rewritten to match the code |
| 15 | `sub_grade` dummies, ND hack `if ['addr_state:ND'] in ...` | unused columns; fragile check | removed; state groups built with `isin()`, which also covers states with no loans |
| 16 | `pd.options.display.max_rows = None` to print 200+ rows | very long outputs | show only the columns that have missing values |
| 17 | `total_rev_hi_lim` filled with the loan amount | it is empty for **every** 2007-2011 loan, so the fill pushed old (riskier) loans into the low-limit categories: the loan-age trap again | analysed, explained, and not used in the models |
| 18 | a separate `dti > 35` category | all of these loans are from 2014 and mostly still running, so they looked safe (+26 points) | merged into the riskiest band (`>22.4`); two very narrow bands merged too |
| 19 | `purpose` groups that were not neighbours in the WoE order | groups must follow the data | regrouped by WoE neighbours; the coefficients now rise steadily |
| 20 | LGD: recovery rate = recoveries ÷ amount **lent**, then multiplied by EAD | recoveries shrank twice, and gross recoveries ignored the collection fee | recovery rate = (recoveries − fee) ÷ amount **owed at default** |
| 21 | PD described without a time period; per-grade PD match called "calibration" | a PD needs a period; the per-grade match is automatic on training data | PD period explained in Part 4; calibration judged on test data only |

---

## 10. Limitations and next steps

**Limitations**
1. `Current` loans count as good, although some will still default, so young loans look safer than they are.
2. The PD has no clean time period ("bad by the time the data was collected"), so the Expected Loss is not a full lifetime loss.
3. Only approved loans are in the data (selection bias).
4. `int_rate` and `grade` are Lending Club's own credit decisions; a lender building its own scorecard would not have them, and using them to set prices would be circular.
5. Recently charged-off loans were still being collected when the data was saved, so their recoveries are understated (LGD overstated).
6. Linear regression does not know that LGD and CCF must lie between 0 and 1; predictions are clipped.
7. One period only (2007-2014), including the financial crisis.

**Next steps**
- Rebuild the PD model on finished loans only (Fully Paid vs Charged Off), which also gives the PD a clear time period.
- Monitor the model on newer loans with the **Population Stability Index (PSI)**.
- Try a challenger model (for example gradient boosting) and compare its AUC with the scorecard.
- Merge neighbouring categories whose points are not significantly different (for example `annual_inc` `<=20K` and `20K-30K`).
