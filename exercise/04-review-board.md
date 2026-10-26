**Language:** [ภาษาไทย](04-review-board.md) | English

# Part B: Architecture review board (approximately 25 min, groups of 4)

> **Lecture link:** AICH4 (reviews need domain expertise; *plausible ≠ appropriate*), Pillar 5 (AI as Chief Analyst, humans as Chief Strategist), JSS (85% of studies keep a human in the loop).
> **Question:** what does a human review board catch that the AI's review missed, and why?

The company's architecture review board meets to approve the R-12 design before the 11.11 sale. One ADR goes to the board: the **architect** presents it, the **AI analyst** presents the AI's review, and two stakeholders bring knowledge the AI never saw.

## 1. Form the board (2 min)

In groups of 4, choose **one** member's ADR to review (`docs/adr/0002-*.md`, plus their `exercise/outputs/3-review.md`). Then assign roles:

| Role | Who | Brings |
|---|---|---|
| **Architect** | the ADR's author: Chayaphon Kultanon | defends the ADR |
| **AI analyst** | Chayaphon Kultanon | reads out the AI's review only, and may not add their own opinions |
| **Product owner** | Nantaporn Phuaphanbun | role card P (business side) |
| **Site reliability engineer (SRE)** | Chatrin Yoonchalard | role card S (operations side) |

Groups of 5 add a **security & privacy officer** (role card X). In groups of 3, the architect also plays the AI analyst.

**Role cards** are in [`docs/review-board/`](../docs/review-board/README.en.md). Each stakeholder reads **only their own card** and keeps it secret until the meeting.

## 2. Meeting (15 min)

| Min | Step |
|---|---|
| 3 | **Architect** presents the decision, the options considered and the main consequences |
| 3 | **AI analyst** reads the AI review's findings and verdict, word for word |
| 7 | **Open discussion.** Each stakeholder raises the concerns from their card, in their own words. The architect responds; anyone may ask the AI analyst "did the AI see this?" |
| 2 | **Decision.** The board votes: accept / accept with changes / reject |

## 3. Record the decision (8 min, together)

**ADR reviewed:** `docs/adr/0002-publish-order-placed-event-via-outbox-and-rabbitmq.md` (transactional outbox + RabbitMQ, from `parm-sw-arch`) by Chayaphon Kultanon
**Board members and roles:** Chayaphon Kultanon (architect, AI analyst), Nantaporn Phuaphanbun (product owner), Chatrin Yoonchalard (SRE)

**Decision:** accept with changes

**Required changes:**
1. **Confirmation page must show points immediately** (PO). Marketing's live 11.11 promise is "see your points the moment you place your order". Show the points on the confirmation page from a synchronous read-only calculation on the order, and keep the async `loyaltyservice` as the system of record. Define point reversal on cancel/refund. Fraud scoring must be able to cancel an order before confirmation, so it stays a synchronous step outside this ADR.
2. **Production readiness and schedule** (SRE, PO). Dashboards, alerts, backups and a runbook for both PostgreSQL and RabbitMQ (about 1 week), a DLQ replay procedure, paging for broker/consumer/relay/email-failure, and trace propagation across the outbox. Request the 3-node broker budget now (2-week lead time, cluster near node limit). Set a go/no-go date with a load test before 11.11, and a fallback if the broker is not ready.
3. **From the AI review.** Relay uses a separate connection pool and holds no lock while waiting for publisher confirms (test: broker down at 200 orders/s, p95 `PlaceOrder` still ≤ 500 ms). Make `emailservice` idempotent. Change the ADR status to `Proposed` until business accepts the "1 minute" exception.

**Who raised what?** List every concern the board discussed.

| # | Concern | Raised by (AI review / architect / PO / SRE) | Was it in the AI's review? | Could the AI have known? Why or why not? |
|---|---|---|---|---|
| 1 | Confirmation page must show points instantly (marketing campaign already live) | PO | No | No. The campaign promise is only on card P, not in the scenario. The AI could have asked what the customer sees as an open question. |
| 2 | Points on cancel/refund; fraud scoring must cancel an order before confirmation | PO | No (the ADR itself lists fraud scoring as an open question) | Partly. The cancel/refund lifecycle is not in the scenario, but a reviewer should ask. The fraud point is already in the ADR and the AI missed it. |
| 3 | Build, test and load-test two new stateful components (PostgreSQL, RabbitMQ) in 3 weeks | PO, SRE | Partly (operability of two stateful components) | No for the timeline. The 3-week window is not in the ADR or the AI's evidence. |
| 4 | No production broker experience (laptops only), 2-person on-call, runbook rule (about 1 week), 3-node broker budget takes 2 weeks | SRE | No (the AI trusted "2 people used RabbitMQ") | No. Skill level and ops rules are tribal knowledge. The AI did flag the missing PostgreSQL skill evidence. |
| 5 | Who is paged, how to replay the DLQ, how to trace "I never got my email" | SRE | Partly (DLQ alert and OpenTelemetry are in the ADR; replay and paging are not) | Yes. These are standard questions the AI could have raised. |
| 6 | Last year's email outage went unnoticed for 20 minutes | SRE | No | No. Incident history is not in the files. It justifies an email failure-rate alert. |
| 7 | Relay may hold a lock/connection and make `PlaceOrder` fail because of the broker | AI review | Yes | Yes. It follows from reading the ADR's own design. |
| 8 | PostgreSQL HA, latency and team skill; "charge then order-write failure" is new, not pre-existing | AI review | Yes | Yes. The ADR's own statements contradict each other. |
| 9 | `emailservice` has no dedup, so a redelivery sends a duplicate email | AI review | Yes | Yes. Visible in the ADR text. |

**The AI's best finding** (one the humans had missed): the relay's `SELECT ... FOR UPDATE SKIP LOCKED` plus waiting for publisher confirms can exhaust the shared connection pool when the broker is slow, so `PlaceOrder` fails because of the broker. That breaks the ADR's central goal, and no card touched it. Its correction of "charge then order-write failure is pre-existing" was also valuable.
**The AI's weakest finding** (generic, wrong or irrelevant): the outbox polling interval, batch size and vacuum/bloat sensitivity point. It is generic, untested, and does not change the decision. The alternative options (CDC, RabbitMQ Streams) were also theoretical, since none addressed the hidden constraints (timeline, skills, budget) that decided the board.

## 4. Individual reflection (≤ 150 words, each student, after class is fine)

Write in [`05-reflection.md`](05-reflection.en.md): after Part A and Part B, which parts of an architect's work would you trust AI with, and which would you keep for humans? Use one concrete example from your board meeting.

> Each group member commits this record to their own repository (copy the same text), so everyone submits a complete repo.