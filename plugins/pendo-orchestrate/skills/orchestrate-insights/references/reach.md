# Reach diagnosis

Answers "why didn't more people get it, who are they, and how long until they get the next message?".

## Premise first

Check the claim before diagnosing it. If someone says an email "isn't sending", read step metrics first. When
sends exist, say how many and since when, then answer the real question.

"Fewer" means fewer sent than entered, unless the user names another base (the audience size, an expected
number). Say which base you used.

## Reads (budget: about 20 calls)

1. `Pendo:getOrchestrateJourney` and `Pendo:getOrchestrateJourneySteps`: schedule, waits, goal, the
   two segment flags, `ignoreThrottling`.
   `Pendo:getOrchestrateJourneyMetrics` with no date range for whether the goal is required: `Pendo:getOrchestrateJourney`
   doesn't return it.
2. `Pendo:getOrchestrateJourneyVisitors` with `enrollmentFilter=all` and `limit` 1 for the uncapped `summary.numVisitors`.
   If that's under 5,000 (or a split brings it there), read the rows (up to 5,000) for:
   - each visitor's `enteredAt`;
   - each visitor's step entry times;
   - where each in-progress visitor is now.

   Note `meta.evaluatedAt` and `summary.truncated`.
3. `Pendo:getOrchestrateJourneyStepMetrics` over the journey's live dates.
4. `Pendo:getOrchestrateJourneyStepVisitors` with `eventType=sent`, then `delivered`, for the email step in question.
5. `Pendo:getSegment` on the entry segment.

## Build a step-by-step count of visitors

Use one column for each stage, and keep the same unit (visitors) all the way down:

1. **Entered**: `Pendo:getOrchestrateJourneyVisitors` with `all` and `limit` 1, `summary.numVisitors`. Read rows only
   if entered is under 5,000 or a split brings it there.
2. **Reached the step**: visitors whose `stepVisits` include the step. If the list is cut even after splitting,
   write **Not counted** and give what the rows you have show, labelled as the first rows by visitor ID.
3. **Sent**: step metrics.
4. **Delivered**: step metrics.
5. **Opened or viewed**: step metrics.

Give each drop its count and its share of the stage before it.

## Explain each drop

Use two lists, never mixed.

**Counted** (from tool fields, or with a counting method from the contract). A number you got by subtracting
one read's count from another's (sent minus completed, active minus sent) is not counted: the two reads count
different visitors and windows. Put it under possible causes, with the read that would count it.
- Visitors still waiting before the step, as of `meta.evaluatedAt`: they are in progress at an earlier step.
- Visitors whose wait ends after the schedule end: they will never reach the step.
- Mutually exclusive branches. When a conditional split sends each visitor down one branch, the first email's sent
  count on each other branch, over the same dates, is a counted lower bound for routing. Report it as counted,
  labelled "emails sent, about one for each visitor", before you list routing as a possible cause. You can't read
  the split rule, so don't say who it chose or why.
- Visitors already marked completed (`completed=true`) before the step, whether by an exit step or a required goal.
- Bounces, from step metrics. A bounce happens after a send, so it never explains a missing send.
**Matched on today's audience** (supporting evidence for a possible cause, not a count of entrants):
- **No email address.** "N of today's audience have no email address" (contract: *Visitors with no email address*).
- **Inactive for over 12 months,** when `reachInactiveVisitors` is false and the audience rule doesn't already
  require recent activity (if it does, skip this and say why). Use `Pendo:buildPendoSegment` for a last
  visit more than 365 days before the journey started, then `Pendo:listVisitors`.
- Say plainly that today's audience isn't the entrant cohort. If it has clearly changed (new entries since,
  or a rule on recent activity), give the number as an indication only.

**Possible causes** (not counted; label each one and name the check that would confirm it):
- **Left the entry segment.** Possible when `retainVisitorsAfterEntry=false` and the rule is one visitors leave by
  acting.
- **Completed through the goal.** Which completed visitors were removed for reaching a required goal. Optional goals remove nobody. Confirm by checking the goal item's events after each visitor's `enteredAt`.
- **Unsubscribed or suppressed** before the send.
- **Throttled at entry.** This explains visitors **missing from "entered"**, never a missing send. Who was held
  is not recorded.
- **Email throttle.** A possible cause of a missing send, effect unknown. Never call it the cause.
- **Third-party email steps** (HubSpot, Marketo, Eloqua). Pendo sees only the hand-off, and its metrics can lag
  by hours.

Never say "today's segment" is the audience the journey had. Segments are evaluated on today's data.

**Also say:**
- *Completed* is not a drop-off. It means visitors who reached an exit step or the journey's end.
- *Entered* is visitors. If the audience is defined by account, or the asker talks about accounts, give the
  accounts reached too (contract: *Accounts*).

## Timing questions

- **Configured time to each message** is the sum of the waits on the path, from `Pendo:getOrchestrateJourneySteps`.
  Explain what each wait means: a wait after a message is how long visitors stay before the next step.
- **Observed lag:** use the contract's *How long until a message is sent* method: daily entries against daily
  sends. Give the share sent the same day and the next day or later.
- Check the schedule's day rules (weekend skips) and say how they move the dates.
- Step entry time is **not** send time. Never call a step-entry time "sent".

## The list

When asked "who" or "can I export them":
- **Where you can write files:** save the IDs to a CSV and give the path. Never paste visitor IDs into a
  shared answer.
- **Where you can't:** give the counts, and say the per-visitor list is not available here.
- **If a list was cut at 5,000:** say so, and say the rows are the first 5,000 by visitor ID, not a sample.

## Cannot answer

- The recorded reason a specific visitor was not sent an email. It is not stored.
- How many visitors the journey throttle held. It is not stored.

Say both plainly when asked.
