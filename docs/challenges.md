<p align="right"><b>🇬🇧 English</b> · <a href="es/retos.md">🇲🇽 Español</a></p>

# Engineering challenges

Real cases from the project, told by their cause and not just their symptom. Each one changed
the way we work.

---

## 1. A green CI that hadn't run a single test

**Situation.** A working branch turned red right after merging a PR that had passed every check.

**Problem.** The job that failed was called "frontend (lint, types, tests, build)", so it looked
like a broken test. It wasn't: **no test ever ran**. It died at step 2 of 5, the formatting
check, over a blank line missing in a documentation file.

**Cause.** The formatting step checks **the whole repository**, but the filter that decides
which jobs run only looked at code. A pure documentation PR matched no path, the job was
skipped, and the gate translated "skipped" as "success".

**What made it invisible.** The filter doesn't compare the same thing on a PR and on a _push_:
on a PR it compares against the base branch; on a _push_, against the repository's default
branch. The content was identical, and the result was green as a PR and red as a _push_.

**Outcome.** The filter now includes everything any verification tool looks at. The general rule
was written down: _if a tool looks at a file, that file must be in the filter._

**Lesson.** Before hunting for the broken test, look at which step the job died in.

---

## 2. A test that failed at night

**Situation.** A test passed all morning and started failing that same night, with nobody
having touched any code.

**Cause.** A test fixture with a hard-coded date was compared against the current date in UTC.
As soon as the UTC clock crossed that date, which in zones west of UTC happens hours before
local midnight, the fixture went from "valid" to "expired".

**Outcome.** Test dates meaning "valid" or "expired" are now computed relative to the moment
of execution, never as literals.

---

## 3. The server decides the price

**Problem.** The client sent the backend the trip's price and distance. A modified client could
pay less.

**Solution.**

- The server computes the distance and applies the official fare table, with minimum and
  maximum limits.
- When charging, the amount comes from the price the server stored when the trip was created,
  **not** from the one the client sends.
- Before charging, it checks that the trip belongs to whoever is paying (IDOR protection).
- If the trip already has an approved payment, a retry returns success without charging again.

All of it with tests covering tampered prices, other people's trips and retries.

---

## 4. MFA that only existed on the screen

**Problem.** The admin panel asked for a second factor at login, but the backend didn't check
it. Someone with an email and password could talk directly to the authentication service and
call the admin API without going through the challenge.

**Solution.** The function that authorizes administrators now verifies, on the server, that the
token comes from a session with a second factor. Without it, the API responds 403.

**Lesson.** A control that only lives in the interface is not a control.

---

## 5. What an error-tracking system could leak

**Situation.** While adding Sentry for observability, a risk showed up that the plan hadn't
considered: by default it attaches the **local variables** of every stack frame, and its data
scrubber only masks keys with an exact name. A local variable named like the payment token
would have traveled in the clear.

**Solution.** Sending local variables was turned off and the personal-data policy was
centralized in a single module, shared by all functions through a Lambda layer.

**Side effect avoided.** The tests clear Sentry's configuration variable before running:
anyone who had it exported for deployment would have sent their tests' errors to the production
project. This was verified by forcing the variable.

---

## 6. Quality measured by history, not by snapshot

**Situation.** CodeScene was adopted as a PR gate. Unlike a linter, it doesn't look for bugs:
it measures **where the code we keep touching hurts**, crossing file health with change
frequency.

**What it taught.**

- A fix dropped a file from 7.89 to 7.87 and blocked a PR that improved seven others. The answer
  wasn't to suppress the rule, but to extract the minimum until the function ended up shorter
  than before the change.
- **Moving code is not improving it.** Pulling almost 800 lines of styles into another file
  improved the original's score and the PR was still rejected: the wall was still the same wall.
- Sometimes the message points the wrong way. A function with **fewer** decision points than
  before got a lower score just because a helper moved from inside the component to outside it
  within the same file. Moral: count the decision points before believing the message.
- Before interpreting the numbers, **the analysis had to be cleaned up**: five author emails for
  two people (including 167 commits with an unreadable email) were distorting code ownership and
  bus factor. It was fixed with an identity-mapping file.

---

## 7. Fewer calls to pay-per-use services

**Mapbox.** Auditing trip tracking showed that every real trip requested a route from a random
point to compute an "arrives in X minutes" that was made-up data. The simulation was restricted
to demos, errors that always fail (the definitive ones) stopped being retried, and two client
implementations were merged into one. The inventory of what's left to optimize was documented.

**Google Places.** Autocomplete without a session token charges for every prediction. The
library generated the token but, in the mode the project uses, didn't send it. It was wired up,
the fields requested for each place's details were limited, and coordinate verification was
restricted to places rather than addresses. The effect on billing still has to be confirmed
once deployed, and it's noted as such.

---

## 8. A pin that lands on the street

**Problem.** The pickup point was the center of the map as-is: it could end up inside a park,
mid-block or on top of a building, and the driver didn't know which street to arrive on.

**Solution.** Every time the map stops, the point is snapped to the nearest drivable street,
like in Uber or DiDi. Mapbox doesn't offer a "nearest point" service, but it does snap every
point of a route to the road network, so a route from the point to itself is requested and the
result is read.

**Uncomfortable decision.** If snapping fails, the pin can't be confirmed. Without it there's no
guarantee it's on a street, so for this feature Mapbox stopped being an optional dependency.

**Engineering detail.** The logic for "only the last requested snap counts" and "don't fall into
a snap-and-move-the-map loop" lives in a class with no React, with its own tests.

---

## 9. Documentation that pointed to the previous state

A cleanup pass found that the design document sent a piece of infrastructure to a dead copy of
a function, that two documents listed as "accepted debt" something that no longer existed, and
that a fallback was described that had already been removed. The rule that came out of it:
**when a fix changes a contract, a name or a behavior, whoever mentions it gets corrected in the
same change.** Documentation that describes a bug is worse than none, because people believe it.

---

## How we work

These cases shaped a workflow applied to every task, however small it looks:

1. Read the CHANGELOG and compare it with the real repository.
2. Make the change.
3. Run the full gate (`verify`).
4. Review the whole diff looking for what the gate doesn't catch: dead code, forgotten debug
   output, decisions that later ones left meaningless.
5. Test what can be tested and **say explicitly what can't** (native behavior, real charges,
   devices).
6. Open the PR. The CHANGELOG is written last, when it describes what actually ended up
   shipping.

And one rule with no exceptions: **never make something merely look like it works.** No bending
a test to what the code does, silencing the rule or lowering the threshold so it stops failing.
