

# 🎯 FSRS Integration – Executive Summary

## 🔹 Goal

Show each question **right before the user is likely to forget it**.

We use:

```
TARGET_RETENTION = 0.92
```

Meaning:

> Schedule next review when predicted recall probability = 92%

---

# 🧠 What FSRS Manages (Memory Engine)

For every **User–Question pair**, we store:

| Parameter      | Meaning                                 |
| -------------- | --------------------------------------- |
| Stability (S)  | How long memory lasts (in days)         |
| Difficulty (D) | How hard this question is for this user |
| State          | New / Learning / Review / Relearning    |
| Due Date       | Next review time                        |

FSRS library automatically:

* Updates S & D
* Calculates optimal interval
* Determines next due date
* Handles state transitions

---

# 🏗 System Architecture Overview

```
                 ┌─────────────────────┐
                 │      User Answers   │
                 └──────────┬──────────┘
                            │ rating (1–4)
                            ▼
                 ┌─────────────────────┐
                 │   Django Backend    │
                 │  (Review Endpoint)  │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │     FSRS Engine     │
                 │ (Python fsrs lib)   │
                 └──────────┬──────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
 Update Stability      Update Difficulty     Compute Interval
        │                   │                   │
        └───────────────────┼───────────────────┘
                            ▼
                   Compute Next Due Date
                            │
                            ▼
                 ┌─────────────────────┐
                 │ Save to Database    │
                 └─────────────────────┘
```

---

# 📦 Database Structure

## 1️⃣ UserQuestionMemory

```
User
Question
Stability
Difficulty
State
Due Date
Lapse Count
Review Count
```

This is the core scheduling table.

---

## 2️⃣ DailyReviewStats

Tracks:

```
User
Date
Reviews Done Today
New Questions Today
```

Used to enforce daily limits.

---

# 🔄 Daily Scheduling Logic

### Priority Rule:

```
1️⃣ Due Questions (highest priority)
2️⃣ New Questions (if capacity remains)
```

---

### Daily Limits:

```
DAILY_REVIEW_LIMIT = 100
DAILY_NEW_LIMIT = 20
```

Rules:

* If due questions ≥ 100 → serve only due
* If due < 100 → fill remaining with new
* Never exceed limits

---

# 📊 Full Flow Diagram

```
User opens exam session
        │
        ▼
Fetch due questions (due <= now)
        │
        ▼
If count < DAILY_REVIEW_LIMIT:
        │
        ▼
Add new questions (max DAILY_NEW_LIMIT)
        │
        ▼
Serve session questions
        │
        ▼
User answers
        │
        ▼
Send rating (1-4)
        │
        ▼
FSRS recalculates:
    - Stability
    - Difficulty
    - State
    - Next Due Date
        │
        ▼
Save updated memory state
```

---

# 📈 What This Guarantees

* Hard questions repeat more often
* Easy questions space out
* Failed questions return quickly
* Intervals grow automatically
* Recall probability stays near 92%
* Workload remains controlled

---

# 🏆 Why This Is Powerful

This transforms your system from:

> “Question & Answer App”

into:

> “Intelligent Memory-Optimized Learning Platform”

It becomes:

* Personalized per user
* Scientifically optimized
* Self-adaptive
* Data-driven
* Scalable

---

# 🧩 One-Slide Ultra Summary (For Non-Technical Audience)

```
We integrate FSRS (Spaced Repetition).

For each user & question:
    → We estimate memory strength.
    → We predict forgetting.
    → We schedule the next review at the optimal time.

Due questions have priority.
Daily workload is limited.
System adapts automatically.

Result:
Maximum retention with minimal effort.
```

