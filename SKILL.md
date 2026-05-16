# Skilled Solution — BuyLead Quality RCA Agent

## TL;DR

BuyLead Quality RCA Agent automatically checks every BuyLead for quality problems before it reaches a supplier. It runs six stages in sequence — fast rule checks first, AI-powered category matching second, language and image checks third, an optional LLM explanation fourth, and a final score last. The output is a 0–100 quality score, a band (EXCELLENT to CRITICAL), a plain-English explanation of what is wrong, and a suggested action for the ops team. The entire pipeline runs in under a second.

---

## The 6 Stages and What Each Does

| Stage | Name | What it does |
|---|---|---|
| 1 | `rule_engine` | Nine fast checks: is the title empty or spammy? are specs filled by the buyer or predicted by AI? is the category assigned at all? is the description a copy of the title? |
| 2 | `mcat_ranker` | Compares the product title against 98,000+ IndiaMart categories using AI embeddings and flags if the assigned category looks wrong. |
| 3 | `nlp_pipeline` | Checks whether the title and specs contradict each other (e.g. title says "Wooden Table" but spec says Material: Plastic). |
| 4 | `image_analyzer` | If an image is attached, checks whether it visually matches the assigned category. Skipped if there is no image. |
| 5 | `llm_narrator` | Only runs when a HIGH or CRITICAL flag is found. Writes a plain-English explanation of what is wrong and why. Also auto-generates a product description if the buyer left it empty. |
| 6 | `scorer` | Collects all findings from stages 1–5 and produces a single quality score out of 100, a quality band, and a suggested action for the ops team. |

---

## Architecture Flow

The pipeline flows top to bottom. Stages 2, 3, and 4 run in parallel after the rule engine completes. The LLM is only called if a HIGH or CRITICAL flag was raised — never blindly.

```
         BuyLead arrives
     (title · specs · category)
               │
               ▼
        ┌─────────────┐
        │ Rule engine │  ← 9 fast checks, no ML, < 5ms
        └──────┬──────┘
               │
       ┌───────┼────────┐
       ▼       ▼        ▼
  ┌─────────┐ ┌─────┐ ┌───────┐
  │  MCAT   │ │ NLP │ │Image  │   (parallel)
  │ ranker  │ │     │ │check  │
  └────┬────┘ └──┬──┘ └───┬───┘
       └─────────┼─────────┘
                 │
                 ▼
         ┌──────────────┐
         │ LLM narrator │  ← only if HIGH/CRITICAL flag raised
         └──────┬───────┘
                │
                ▼
           ┌────────┐
           │ Scorer │  ← 0–100 score + quality band
           └────────┘
                │
                ▼
           RCA Report
  (score · flags · MCAT suggestions
   narrative · suggested action)
```

---

## Quality Bands

| Band | Score | What it means | Action |
|---|---|---|---|
| EXCELLENT | 80–100 | Clean lead, correct category, buyer-filled specs | Auto-approve and route |
| GOOD | 60–79 | Minor issues, still usable | Approve with note |
| MEDIUM | 40–59 | Noticeable gaps in specs or category | Review before routing |
| POOR | 20–39 | Multiple problems, risky to route | Manual review required |
| CRITICAL | 0–19 | Spam, empty title, or completely wrong category | Block — fix first |

---

## What Gets Flagged

Each flag has a severity (CRITICAL / HIGH / MEDIUM / LOW), the exact evidence that triggered it, and a suggested fix.

| Flag | Severity | What triggered it |
|---|---|---|
| `EMPTY_TITLE` | CRITICAL | Title is missing or under 3 characters |
| `SPAM_TITLE` | CRITICAL | Title matches a spam word list (test, aaa, xyz…) |
| `MCAT_UNMAPPED` | CRITICAL | No category assigned (MCAT ID = 0) |
| `MCAT_MISMATCH` | HIGH | AI ranker's top category differs from the assigned one |
| `NO_BUYER_FILLED_ISQS` | HIGH | Buyer answered 0 spec questions — all values are AI-predicted |
| `B2C_DESCRIPTION` | HIGH | Description shows personal-use intent in a B2B marketplace |
| `MISSING_ISQ_FIELDS` | MEDIUM | Required spec fields for the assigned category are blank |
| `SPEC_TITLE_CONTRADICTION` | MEDIUM | Title and a spec field contradict each other |
| `TITLE_TOO_LONG` | LOW | Title is over 15 words — hurts search relevance |

---

## Validated on Real Cases

**Case 1 — wrong category caught**
A buyer posted "Plastic 4dx JCB Excavator Toy" under *JCB Excavators* (heavy machinery). The MCAT ranker found *Toy Vehicles* as a 91% confidence match and raised `MCAT_MISMATCH`. The buyer had also answered 0 of 5 spec questions. Final score: 38/100 — POOR. The lead was blocked from machinery suppliers and flagged for remapping.

**Case 2 — clean lead confirmed**
A buyer posted "Cotton Carry Bags" under *Cloth Bags* with buyer-filled specs (Bag Type, Material, Quantity). The ranker confirmed *Cloth Bags* at 94% confidence. No flags raised. Final score: 82/100 — EXCELLENT. Auto-approved.

---

## Why This Is a Skilled Solution

**Replaceability.** Swap the LLM endpoint in `06_llm_reasoner.py` and nothing else changes. Update the MCAT dump and the ranker picks it up at load time. Add a new check module and register its score impact in the scorer — no other file changes.

**Demonstrability.** Stages 1–4 run without any LLM or internet access. You can test the rule engine or MCAT ranker in isolation with a single JSON input. The Gradio demo ships four pre-loaded examples covering four distinct failure modes.

**Auditability.** Every RCA report carries the exact evidence that produced it. A flag like `MCAT_MISMATCH` says *"AI ranker top candidate: Toy Vehicles at 91% confidence"* — not just *"category may be wrong"*. The ops rep sees the reasoning, not just the conclusion.

---

## Where to Look

- `02_rule_engine.py` — all nine rules, typed flag objects, ISQ source scoring
- `03_mcat_ranker.py` — FAISS search + reranking against 98k categories
- `07_scorer.py` — weighted scoring model; change weights here and everything updates
- `08_rca_agent.py` — the orchestrator; shows the full pipeline and the `use_llm` gate
- `09_demo_gradio.py` — interactive UI with single-lead and batch-upload tabs

Project Link : https://4b13d38ae3049d59b1.gradio.live/
