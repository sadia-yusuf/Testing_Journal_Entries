# Journal Entry Testing at Population Scale

Testing **84,106 journal entries** against eight audit procedures — and measuring the detection rate against a known answer key.

Built three ways: Python for verification, Excel Power Query and Power Pivot for implementation, Alteryx Designer for reviewability and productionisation.

---

## Result

| Measure | Value |
|---|---:|
| Journal entries tested | 84,106 |
| Line items processed | 168,212 |
| Test hits raised | 558 |
| Unique entries flagged | 542 |
| Exception rate | 0.64% |
| **Anomalies planted** | **124** |
| **Anomalies detected** | **124 (100%)** |
| Anomalies missed | 0 |
| False positives | 418 |
| Precision | 22.9% |

Detection by anomaly type:

| Anomaly type | Planted | Caught | Rate |
|---|---:|---:|---:|
| Below approval threshold | 26 | 26 | 100% |
| Non-business day posting | 22 | 22 | 100% |
| Back-dated | 18 | 18 | 100% |
| Narration keyword | 18 | 18 | 100% |
| Round number above materiality | 14 | 14 | 100% |
| Rare user | 9 | 9 | 100% |
| Sequence gap | 9 | 9 | 100% |
| Rare account pairing | 8 | 8 | 100% |

---

## Why this project is different

Most analytics portfolio work can report what a workflow **found**. Almost none can report what it **missed**, because the ground truth is unknown.

The ledger here is synthetic and generated with anomalies deliberately planted and recorded in an answer key. Detection becomes a measurement rather than an assertion — and the false positive rate can be reported honestly instead of tuned away.

---

## The problem being solved

Journal entry testing is conventionally performed on a sample — often twenty-five entries drawn from tens of thousands. That is 0.03% coverage. The remaining population goes untested and its exception rate is unknown.

SA 240 requires journal entry testing as a mandatory response to the risk of management override. Manual journal entries are the standard mechanism for that override, because they bypass the controls built around ordinary transaction processing.

This project tests the entire population against eight procedures, then ranks exceptions by how many independent procedures flagged each entry.

---

## The eight procedures

| Ref | Procedure | Exceptions |
|---|---|---:|
| T1 | Round-number manual postings at or above materiality | 14 |
| T2 | Manual entries posted on weekends or public holidays | 29 |
| T3 | Entries by users posting fewer than ten times in the year | 9 |
| T4 | Entries posted more than seven days after effective date | 18 |
| T5 | Manual amounts between 94% and 100% of the approval threshold | 445 |
| T6 | Debit/credit account pairings occurring fewer than five times | 8 |
| T7 | Narrations containing review-trigger words | 26 |
| T8 | Gaps in the journal entry identifier sequence | 9 |

Benford's Law was deliberately excluded — it assumes naturally occurring numbers, and a corporate ledger is full of negotiated values.

Full reasoning for each procedure, including thresholds and limitations, is in [`docs/Test_Objectives_and_Significance.docx`](docs/).

---

## Two findings worth reading

### The completeness check nobody runs

Thirty-six line items across eighteen entries carry posting dates **after** the financial year end — year-end accruals with effective dates in March, posted in April and May.

Entirely normal. But it means **any analysis scoping this population on posting date alone silently loses eighteen genuine year-end entries**, and produces a clean-looking result on an incomplete population. That is worse than no analysis, because it carries false confidence.

The population is therefore scoped on effective date, with posting date retained as a test attribute.

### 2,231 exceptions is a failed test

T5 as first designed returned **2,231 exceptions to identify 26 genuine items** — a precision of 1.2%. Of those, 1,786 were system-generated batch postings.

Four designs were compared:

| Design | Exceptions | Caught | Missed | Precision |
|---|---:|---:|---:|---:|
| 94–100%, all entries | 2,231 | 26 | 0 | 1.2% |
| **94–100%, manual only** | **445** | **26** | **0** | **5.8%** |
| 98–100%, manual only | 130 | 7 | 19 | 5.4% |
| 90–100%, manual only | 720 | 26 | 0 | 3.6% |

Restricting to manual entries improved precision fivefold at no cost in detection. Narrowing the band instead lost nineteen of the twenty-six.

**The band was never the problem — the entry type was.** Two thousand exceptions is a failed design, because a list nobody works through is not a control.

---

## Repository structure

```
journal-entry-testing/
├── README.md
├── notebooks/
│   ├── 01_generate_ledger.ipynb        generates the ledger and answer key
│   └── 02_reconnaissance.ipynb         runs all eight tests, sets target figures
├── excel/
│   ├── journal_entry_testing.xlsx      queries, data model, dashboard
│   ├── powerquery_source.txt           all 16 M queries, readable
│   ├── dax_measures.txt                11 Power Pivot measures
│   └── screenshots/
├── data/
│   ├── trial_balance.csv               34 accounts
│   ├── answer_key.csv                  124 planted anomalies
│   └── general_ledger_sample.csv       first 5,000 lines
└── docs/
    ├── methodology_memo.md             scope, procedures, thresholds, limitations
    └── Test_Objectives_and_Significance.docx
```

The full 23 MB ledger is not committed. The generator uses a fixed seed, so running `01_generate_ledger.ipynb` reproduces it byte-identically.

---

## Reproducing the results

```bash
pip install numpy pandas
jupyter notebook notebooks/01_generate_ledger.ipynb   # run all
jupyter notebook notebooks/02_reconnaissance.ipynb    # run all
```

Then open `excel/journal_entry_testing.xlsx` and refresh all queries. Update the file paths in the source queries to your local data folder first.

Both routes produce the same 124 detections from the same 84,106 entries.

---

## Method

Seven steps, carried over from statutory audit rather than from analytics practice:

1. Scope the assertion, not the deliverable
2. Prove completeness before analysing anything
3. Tie to the source of truth and document any difference
4. Build the logic one movement at a time, never netted
5. Design the exception, not the report
6. Build the audit trail as you go
7. Re-run on the next period's data before calling it done

The standard applied throughout: **a reviewer who has never spoken to the preparer should be able to re-perform this work and reach the same result.**

---

## Tools

**Python** — pandas, NumPy. Data generation and independent verification of every target figure.

**Excel** — Power Query for the eight procedures and consolidation, Power Pivot for the star-schema model and eleven DAX measures, pivot tables and slicers for the dashboard.

**Alteryx Designer** — same eight procedures as parallel branches on a single canvas, packaged as an Analytic App. *(In progress.)*

---

## Limitations

- **Synthetic data.** Real ledgers have messier narrations and inconsistent user conventions. Detection rates on real data would be lower.
- **Two-line entries only.** Compound entries with multiple debits and credits would require the account-pairing procedure to be redesigned.
- **Detection measured against planted anomalies only.** The procedures may fail to detect patterns not represented in the answer key.
- **No control testing.** This addresses recorded entries, not the design or operating effectiveness of controls over them.
- **Thresholds are dataset-specific.** Materiality, the approval limit and the rare-user cut-off would each be set from engagement circumstances.
- **Analytics extends coverage, not judgement.** Every exception raised is an item requiring enquiry, not a conclusion.

A well-constructed improper entry — posted mid-week by a routine user, for an untidy amount, to a routine account pairing, with a plausible narration — would pass all eight procedures.

---

## About

Built by **Sadia Yusuf**, Chartered Accountant, with eight years of statutory audit experience across banking, infrastructure and financial institutions.

I build the procedures I previously performed by hand.

Synthetic data generated for this project. No client data, client methodology or engagement material is used. Prepared as portfolio work, not as an audit deliverable.

sadiayusuf23@gmail.com
