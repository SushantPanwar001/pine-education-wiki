# Pine Education Backend

The backend for a computer-based test (CBT) platform serving iOS and Android clients. Students take exams, admins manage content and analytics, and all assessment purchases are mediated through a virtual currency called Pine Coin.

## Language

### Assessment

A user's timed attempt at an exam. Created when a user starts an exam, transitions through statuses (`IN_PROGRESS`, `COMPLETED`, `ABANDONED`), and is automatically graded on submission or timer expiry.
_Avoid_: attempt, test, quiz

### Exam

A configured set of sections containing questions, with timing and navigation rules. Created and managed by admins. Has a configurable price in Pine Coins.
_Avoid_: test, paper

### Section

A timed grouping of questions within an exam. Each section has its own marking scheme (marks per correct/wrong answer), optional duration, and question count to serve per attempt.
_Avoid_: module, part

### Question

A single problem within a section. One of three types: MCQ (single correct), MSQ (multiple correct), or NAT (numerical answer type). Content is Markdown with LaTeX support. Images are stored in Cloudflare R2.
_Avoid_: item, problem

### Response

A user's answer state for a specific question within an assessment. Tracks visit status, input, marks awarded, and correctness after grading.
_Avoid_: answer (use "answer" only for the user's input, not the record)

### Pine Coin

Virtual currency used to purchase assessments. Conversion rate: 1 INR = 5 Pine Coins. Non-refundable once deducted. Tracked via a transaction ledger.
_Avoid_: credit, token, point

### Bundle

An admin-created coin pack purchasable with real currency. Contains a fixed number of Pine Coins at a set price in INR, with optional bonus coins. The primary mechanism for users to acquire Pine Coins (payment gateway integration pending).
_Avoid_: package, plan, subscription

### Coin Transaction

A ledger entry recording every Pine Coin movement (credit or debit) with a running balance, type, and description. Provides full auditability.
_Avoid_: transaction (too generic — use "Coin Transaction" for clarity)

### Question Batch

A group of questions extracted from an uploaded PDF via Google Gemini AI. Goes through a review workflow where admins accept, reject, or skip individual questions.
_Avoid_: import, upload (those are the actions, not the entity)

## Relationships

- An **Exam** contains one or more **Sections**, each with a configurable marking scheme
- A **Section** contains many **Questions** (MCQ, MSQ, or NAT)
- A user starts an **Assessment** by paying the exam's price in **Pine Coins** (atomic deduction)
- An **Assessment** belongs to one **User** and one **Exam**, and contains many **Responses** and **Section Attempts**
- A **Response** links one **Question** to one **Assessment** with the user's input and grading result
- A **Bundle** is purchased with INR (via payment gateway, pending) and credits **Pine Coins** to a user
- Every **Pine Coin** movement creates a **Coin Transaction** ledger entry

## Example dialogue

> **Dev:** "When a user starts an **Assessment**, do we deduct **Pine Coins** immediately?"
> **Domain expert:** "Yes — the deduction and assessment creation happen in one atomic transaction. No refunds, even if the user abandons."
>
> **Dev:** "What if they don't have enough coins?"
> **Domain expert:** "The request is rejected with an insufficient balance error. They need to purchase a **Bundle** first."
>
> **Dev:** "How does the admin review AI-extracted questions?"
> **Domain expert:** "They upload a PDF, which creates a **Question Batch**. Gemini extracts questions, then the admin accepts, rejects, or skips each one."

## Flagged ambiguities

- "subscription" was previously used but has been **dropped** in favor of the Pine Coin model. The `subscriptionTier` field on the user table is vestigial and should be removed in a future migration.
- "answer" is ambiguous — it can mean the user's input (use "answer" informally) or the database record (use **Response**).
- "transaction" is ambiguous in financial contexts — use **Coin Transaction** for Pine Coin ledger entries specifically.
