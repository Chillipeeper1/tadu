<p align="right"><b>🇬🇧 English</b> · <a href="es/decisiones.md">🇲🇽 Español</a></p>

# Architecture decisions

Seven decisions with their context and, above all, **what was given up** when making them. This
is a condensed version of the project's ADRs.

| #   | Decision                                                      |
| --- | ------------------------------------------------------------- |
| 1   | Serverless backend instead of a traditional server            |
| 2   | DynamoDB with a single table                                  |
| 3   | Automatic FIFO assignment, with no accept/reject offer        |
| 4   | Tokenize the card on the client                               |
| 5   | Context API instead of Redux or Zustand                       |
| 6   | Google Maps to draw, Mapbox to compute routes                 |
| 7   | OTA updates tied to the app version                           |

---

## 1. Serverless instead of a server

**Context.** A two-person team and a very uneven load: spikes when someone requests a taxi and
silence the rest of the time. Production infrastructure was needed without spending time
operating it.

**Decision.** Lambda + API Gateway + DynamoDB, defined as code with AWS SAM.

**Given up.** Cold starts, clumsier local debugging and dependence on AWS.

**Gained.** Automatic scaling, no servers to maintain and pay-per-use pricing.

## 2. DynamoDB with a single table

**Context.** The access patterns were few and well known: driver queue per base, history per
passenger, history per driver, and detection of inactive sessions.

**Alternatives.** Several DynamoDB tables, or a serverless relational database.

**Decision.** One table with four secondary indexes, one per pattern.

**Given up.** The model has to be planned up front, there are no joins, and ad hoc reports are
awkward without exporting the data.

**Gained.** Predictable latency at any scale and atomic transactions across different entities
without leaving the table, which is exactly what trip assignment needs.

## 3. FIFO assignment, no offers

**Context.** Ride-hailing apps usually show the request to the driver and let them accept or
reject. At a taxi stand, the driver is already physically waiting for their turn.

**Decision.** When a trip is requested, the system assigns directly to the first driver in the
queue.

**Given up.** The driver can't decline a specific trip. Their only way out is to check out.

**Gained.** The offer timer, reassignment on rejection and intermediate states disappear. The
flow is shorter and has fewer places to fail.

## 4. Tokenize the card on the client

**Context.** Handling card numbers in our own backend requires full PCI certification, which is
disproportionate for an MVP.

**Decision.** The app gets a token directly from the payment processor and the backend only
receives that token.

**Given up.** Part of the payment flow stays on the client, so the server can't trust what the
client says: the amount is always recalculated on the server.

**Gained.** The simplest compliance level (SAQ-A), and no card data passes through our own
servers.

## 5. Context API instead of Redux or Zustand

**Context.** Session, profile and trip state had to be shared between screens.

**Decision.** React contexts and local state, with no state library.

**Given up.** State-history debugging tools, and nested contexts can get verbose if the app
grows a lot.

**Gained.** One dependency fewer and less surface to maintain.

## 6. Google Maps to draw, Mapbox for routes

**Context.** Two different needs: showing a map users already know, and computing routes,
distances and snapping pins to the street.

**Decision.** A mixed provider setup.

**Given up.** Two SDKs, two accounts, two keys and double mandatory attribution.

**Gained.** The cost of computing routes stays low at MVP volumes, and the interface keeps using
the map users are already familiar with.

**Follow-up.** Calls to both services were later measured and reduced with no visible change for
the user (see [`challenges.md`](challenges.md)).

## 7. OTA updates tied to the app version

**Context.** The goal was to ship JavaScript fixes without rebuilding the native binary, and to
decide which updates are compatible with which builds.

**Alternative.** Compute a fingerprint of the native code on every build.

**Decision.** Compatibility by app version, which is bumped deliberately.

**Given up.** If someone adds a native module and forgets to bump the version, an update could
reach a binary without that module and fail at runtime.

**Gained.** Explicit, predictable control over when compatibility is cut.
