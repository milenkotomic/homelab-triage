# Debugging under a budget

Shared reasoning rules for triage skills. Copied, not linked, into each skill
that uses them; keep the copies in sync by hand.

## Root cause before fix

A fix proposed before the cause is known is a guess with side effects. Restarting
the noisy container makes the symptom go away and destroys the evidence. Name
the cause, show the number that proves it, then suggest the fix.

A root cause explains the symptom, its timing and its scope. If the candidate
explains the symptom but not why it started at 14:05, or why only one host is
affected, it is a contributing factor, not the cause. Keep looking or report it
as the best lead with medium confidence.

## Do not anchor

The place the user noticed the problem is where the symptom surfaced, not
necessarily where it started. A slow photo app may be a full disk, a DNS
resolver that is down, or a neighbour container eating memory. That is why the
wide pass is mandatory: it is cheap compared with drilling into the wrong
service for ten queries.

Write the starting point down as "where you came in" and forget it until the
wide pass is done.

## Change is the signal

Every system has steady errors: a check that always fails, a component that logs
warnings as errors. They were there yesterday and last week. What matters is
what changed: compare the window with a baseline of the same length (same hour
last week by default, or the hour before a deploy). Report steady noise
separately so nobody chases it next time.

## One hypothesis at a time

Before each drill query, write the hypothesis, what observation would make it
false, and the query. Then run it and record a verdict. This prevents the most
expensive failure: running queries "to see what's there" and reading meaning
into whatever comes back.

Prefer the query that can rule a hypothesis out over the one that can only
support it. "Disk full on host A" is ruled out by one number; "the app is buggy"
cannot be ruled out by anything, so it is not a hypothesis yet.

## Correlation, timing, scope

- Timing: the cause starts before or with the symptom, never after.
- Scope: a host-level cause affects every service on the host; if only one
  service is affected, look at that service or its dependencies.
- Correlation: two signals that moved together are both suspects until one is
  shown to drive the other (the dependency direction, or which moved first).

## The budget

The budget exists because unbounded investigation drifts. Spend it roughly as:
a third on the wide pass, the rest on drills. When it runs out, stop and report
honestly: the best lead, what was ruled out, and the next query you would have
run. A clear "not found, here is what's left" is more useful than a confident
wrong answer.

## Confidence

- High: the cause is confirmed by a number, matches timing and scope.
- Medium: it matches two of the three, or the evidence is indirect.
- Low: it is the most likely of the suspects left, nothing more.
