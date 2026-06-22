# Metric Definitions

This document defines every KPI shown on the dashboard — the exact formula, the
grain it's calculated at, and known caveats. The goal is a single source of
truth so anyone (analyst, stakeholder, or future me) can trust what a number
means without re-deriving it.

---

## Self-Serve Adoption Rate

**Definition:** The share of all tickets handled through self-serve channels
(Chatbot, Community Forum) rather than agent-assisted channels (Chat, Email,
Phone).

**Formula (Tableau calculated field):**
```
SUM(IF [is_self_serve] THEN 1 ELSE 0 END) / COUNT([ticket_id])
```

**Grain:** Ticket-level, aggregated to whatever dimension is on the viz
(e.g., by month, by product area).

**Caveats:**
- Counts a ticket as "self-serve" based on the *channel it was raised in*,
  not whether it was ultimately resolved there. A self-serve ticket that gets
  escalated to an agent still counts toward adoption, not resolution.
- Does not account for repeat contacts — a customer trying self-serve twice
  for the same issue is counted as two tickets.

---

## Automated Resolution Rate

**Definition:** The share of all tickets fully resolved by automation
(bot/community self-serve flow) with zero human agent involvement.

**Formula:**
```
SUM(IF [resolved_by_automation] THEN 1 ELSE 0 END) / COUNT([ticket_id])
```

**Grain:** Ticket-level.

**Caveats:**
- This is a stricter metric than adoption — a ticket can be *raised* via
  self-serve but still count as a "miss" here if it escalates or goes
  unresolved.
- Should always be read alongside **Escalation Rate** and **Reopen Rate** to
  catch cases where automation looks successful on paper but the issue
  wasn't actually solved.

---

## Self-Serve CSAT

**Definition:** Average customer satisfaction score (1–5) from customers who
responded to a post-interaction survey after a self-serve resolution.

**Formula:**
```
AVG(IF [is_self_serve] AND [csat_responded] THEN [customer_satisfaction_rating] END)
```

**Companion metric — Agent-Assisted CSAT:**
```
AVG(IF NOT [is_self_serve] AND [csat_responded] THEN [customer_satisfaction_rating] END)
```

**Grain:** Ticket-level, survey-respondents only.

**Caveats:**
- CSAT response rate is ~32–45% in this dataset, meaning the score reflects
  only the subset of customers who chose to respond — a known source of bias
  in any CSAT metric (response bias tends to skew toward strong opinions in
  either direction).
- Should not be read in isolation from adoption or resolution rate — rising
  adoption with falling CSAT is a meaningfully different story than rising
  adoption with stable CSAT.

---

## Cost Deflection

**Definition:** The cost difference per ticket between self-serve and
agent-assisted channels, used to estimate savings from shifting volume to
automation.

**Formula:**
```
Avg Cost – Self-Serve = AVG(IF [is_self_serve] THEN [cost_per_ticket_usd] END)
Avg Cost – Agent       = AVG(IF NOT [is_self_serve] THEN [cost_per_ticket_usd] END)
```

**Grain:** Ticket-level; can be sliced by `customer_segment` for a more
precise view (enterprise tickets carry a materially higher agent-cost
multiplier, reflecting dedicated agent/CSM handling).

**Caveats:**
- This is a *blended* cost view, not a fully loaded cost model — it doesn't
  include fixed infrastructure costs for maintaining the bot/self-serve
  platform itself.
- Escalated tickets carry both a self-serve cost *and* an added agent cost,
  which is reflected in the data but easy to double-count if you're not
  careful when summing total cost by channel.

---

## Supporting / Diagnostic Metrics

| Metric | Formula | Why it matters |
|---|---|---|
| **Escalation Rate** | `SUM(IF [resolution_outcome]="escalated_to_agent" THEN 1 ELSE 0 END) / COUNT([ticket_id])` | High escalation erodes the savings claimed by automated resolution rate |
| **Reopen Rate** | `SUM(IF [reopened] THEN 1 ELSE 0 END) / COUNT([ticket_id])` | Flags false-positive "resolutions" — especially important to check by channel |
| **Open/Pending Backlog %** | `SUM(IF [ticket_status]!="Closed" THEN 1 ELSE 0 END) / COUNT([ticket_id])` | Operational health signal, independent of resolution-flavored metrics |
| **First Response Time** | `AVG([first_response_time_minutes])`, by channel | Speed story — usually the cleanest self-serve win |
| **Time to Resolution** | `AVG([time_to_resolution_minutes])`, closed tickets only | Pairs with FRT to show full lifecycle speed |

---

## Ownership & Update Cadence

- **Owner:** Analytics (this project is a portfolio sample; in a real team
  setting this section would name the accountable analyst/team).
- **Source data grain:** One row per `ticket_id`.
- **Refresh cadence (proposed):** Daily for the live dashboard; this document
  should be reviewed any time a metric formula changes, with a changelog
  entry below.

## Changelog

| Date | Change |
|---|---|
| 2026-06-20 | Initial metric definitions published alongside v1 dashboard |
