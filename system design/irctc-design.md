Irctc system design

ideas below

1. schedule of trains
2. book a single seat or multiple seats in a given train on a given date maybe given time bcoz some train run multiple times a day
3. train has ROUTE, Arrangement of seats.

4. ROUTE is STOPS , time, date maybe calendar is for 3 months advance booking

5. transaction where parallel seat booking --> amtomicity gurantee

6. cancellation --> no bookings

start with schedule of trains

a service which gives me train schedules

NOTE: when in doubt always look for what kind of query will be done on DB
I am thinking relational datable because of tine range query

Let me estimate the number of trains and station and daily trains stopage bcoz i am thining of storing let say A to B to C as two record one from a to b and one from b to c as TRain details data is likely constant may be we can use it in memory ,let me calculate

what i know is , i watched a tv show which said more indians travel each day via train than population of australia which will be ball park 23 million or 2.3 crore.

now for calculation let me make 2 crore passengers per day.
TRAINS probably 13 thousand per day
20000000/ 10000
per train 2000

24 hrs -> 30 mins each stop so roughly 50 stops
13000 \* 50 = 650,000 --> this many records which store data of every train which is below 1Million , SQL can handle and they are cheap

id -> 8 bytes
source and dest -> 64 _ 2
date times -> 8 _ 2

total for each row = 8 + 128 + 16 = 152 bytes ~150 bytes

so total = 6,50,000 \* 150 = 650 + 325 = 9,75,00,000 bytes = 0.09gb (may be use in memory)

schedule
tains id
source
destination
expect dept time
expected arrival time

Train has
id
number of seat
seat name

query : start time filter [start, end] binary search on EST time of dept.

also source and destination based filter then start and end

// book a seat query , whole route of the train

getSchedule(ID, from timestamp, to timestamp)
// if train has multiple date travel

// for every source dest. i need to give seats available assuming one seat can be given from source to dest not like if you I have to go A to C via B i can opt different seat for A to B and again different for B to C.

so for each ticket booking i will decrement seat count

and i will also need a Ticket object which has

Train
user
seat

but not enough information what about someone board and exit in between whole journey

i think i will need Booking

train id
source
dest
seat_left

thing is implementing seats based on location is a problem like for station B train is full but then for next station C train has some empty seats , i will need station wise seat availablity

for example

A -> B -> C -> D -> E
1 1 1 2 -->seat availablity
again a booking from A to D
0 0 0 1

now if someone wants to book from D to E they can

let me brute force if
13000 trains each train has let say 500 passengers
so 13000 \* 500 = 6500000 na 6.5 million if start to end of passengeris same as train is not optimal to store directly

in booking table i can lock every station from source to destination for a passenger and decrease the count

to get all station btw source to end
i can create a ROUTE table
which is like a graph

Train id
destination
timestamp

so i can check and lock all rows in booking for train id x and its destination where i will check and block rows which come betweem booking source station time and destination time
and then unblock it. sound good -> exclusive locks

but what happends if A to B seat is different and B to C seat is different then assumption for making one seat per passenger fixed fails.

oh for
a b c
2 2 2
if a to c
1 1 2 solves this problem exclude the final destination

ticket will also take care of source and dest also

what about optimistic lock instead of pessimistic lock

like decrement the counter in ROUTE optimistically if it happens good other wise will see, like reduce A , b seats for a to c travel where we save initial seat avaiablity and compare with final lock other wise abort, bcoz if some one tried to book b to c at the same time we can handle that case here and reject the a to c travel ticket if seats full pop up

pessimistic lock is better to understand
i think too much stress on optimistic

what about tatkal , hotel booking , coach type....

times up
i had 45 mins to think about

// below is claude response to my thought process and what was wrong

# IRCTC System Design — Attempt Review & Correct Approach

> **Context**: This is a 45-minute system design attempt reviewed and documented for learning.  
> The goal is to capture what thinking was right, what was missing, and what the correct approach looks like — so I can return to this when preparing for interviews.

---

## My Original Attempt (Summary)

- Estimated 2 crore daily passengers → ~13,000 trains → ~50 stops → 650k route records
- Calculated storage per row (~150 bytes) → ~0.09 GB → fits in memory
- Modeled route as `(train_id, source, destination, dep_time, arr_time)` pairs
- Identified the segment-locking problem for seat availability
- Raised pessimistic vs optimistic locking and chose pessimistic
- Mentioned tatkal and cancellation at the end as afterthoughts

---

## What I Got Right

### 1. Capacity estimation before schema design

Starting with "how many passengers → how many trains → how many records" before touching any schema is exactly right. Most candidates jump straight to tables. I calculated:

```
2 crore passengers/day
→ ~13,000 trains
→ 50 stops per train (every 30 mins over 24hrs)
→ 13,000 × 50 = 650,000 route records

Per row: 8 (id) + 128 (src+dst) + 16 (timestamps) = ~150 bytes
Total: 650,000 × 150 = ~97 MB → fits in memory (Redis or app cache)
```

This is good. Estimating storage before choosing DB is the right order of thinking.

---

### 2. Route segmentation model

Storing A→B→C as two records `(A→B)` and `(B→C)` rather than one `(A→C)` is correct. It maps cleanly to a stop-by-stop model and enables range queries. This is close to how production railway systems actually model routes.

---

### 3. Seat availability per segment — the hardest part

The key insight I got right:

```
Station:  A    B    C    D    E
Seats:    1    1    1    2         ← available between each pair

Booking A→D:
          0    0    0    1         ← decrement A,B,C but not E (exit point)
```

This means someone can still book D→E because seat is free from D onwards. This is non-trivial. Most candidates miss this entirely and model seat availability as a single integer per train, which breaks for mid-route boarding.

---

### 4. Choosing the right DB with the right reasoning

I said: _"relational DB because of time range query"_ — and that's the right reasoning chain. Choosing a DB based on query pattern, not familiarity, is the correct approach.

---

### 5. Pessimistic vs optimistic locking — raised both

I described both, understood the tradeoff (optimistic has retry cost, pessimistic has lock contention), and picked pessimistic as simpler and more predictable. That's a valid call for an initial design. In a follow-up I could extend to optimistic with retry for high-throughput scenarios.

---

## What Was Missing or Wrong

### 1. No API design

I never sketched a single endpoint. An interviewer expects at least:

```
GET  /trains/search?from=DEL&to=MUM&date=2024-03-15
GET  /trains/:trainId/availability?date=...&src=...&dst=...
POST /bookings
DELETE /bookings/:pnrId
GET  /bookings/:pnrId
```

API design is not optional — it forces clarity on what data flows where and catches gaps in the schema early.

---

### 2. Ticket/Booking table was never finalized

I mentioned `train, user, seat` but never wrote a complete schema. Here is what a proper ticket table looks like:

```sql
CREATE TABLE tickets (
    ticket_id     BIGINT PRIMARY KEY,
    pnr           VARCHAR(10) UNIQUE NOT NULL,
    user_id       BIGINT NOT NULL,
    train_id      BIGINT NOT NULL,
    journey_date  DATE NOT NULL,
    src_station   INT NOT NULL,
    dst_station   INT NOT NULL,
    seat_number   VARCHAR(10),
    coach_type    ENUM('SL', '3A', '2A', '1A', 'CC'),
    status        ENUM('CONFIRMED', 'RAC', 'WAITLIST', 'CANCELLED'),
    booked_at     TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (train_id) REFERENCES trains(train_id)
);
```

The PNR — the thing IRCTC is literally named after (Passenger Name Record) — never appeared in my design. That's a significant miss.

---

### 3. No waitlist or RAC logic

IRCTC has three seat states: CONFIRMED, RAC (Reservation Against Cancellation), and WAITLIST. I mentioned tatkal only at the end. The correct approach is to design for these from the start because they affect the booking flow significantly:

```
CONFIRMED  → seat_left > 0
RAC        → seat_left == 0 but rac_quota > 0
WAITLIST   → both exhausted, waitlist position assigned

On cancellation:
  WL/1 → RAC → CONFIRMED (cascading upgrade)
```

This cascade logic should be handled by a background job or a DB trigger, not inline in the booking transaction.

---

### 4. Locking mechanism lacked precision

I said "lock all rows between source and destination" but didn't specify _how_. The correct implementation is:

```sql
BEGIN;

-- Lock only the affected segments for this train+date+coach
SELECT * FROM seat_inventory
WHERE train_id = ?
  AND journey_date = ?
  AND stop_order >= ? AND stop_order < ?  -- src_order to dst_order exclusive
  AND coach_type = ?
FOR UPDATE;

-- Check seats_left > 0 for ALL locked rows
-- If yes, decrement
UPDATE seat_inventory SET seats_left = seats_left - 1
WHERE train_id = ? AND journey_date = ?
  AND stop_order >= ? AND stop_order < ?;

-- Insert ticket record
INSERT INTO tickets (...) VALUES (...);

COMMIT;
```

The `FOR UPDATE` is the key — it locks those specific rows for the duration of the transaction, blocking concurrent bookings for overlapping segments.

Deadlock risk: if two bookings lock segments in opposite directions (A→C and C→A simultaneously), they can deadlock. Fix: always acquire locks in ascending `stop_order` order.

---

### 5. No payment failure handling

What happens if:

- Payment gateway times out after seat is locked?
- DB write succeeds but payment fails?

The standard pattern is a **reservation timeout**:

```
Book request → create ticket with status = PENDING
             → hold seat for 10 minutes
             → if payment confirmed → status = CONFIRMED
             → if timeout → release seat (cron job or TTL-based cleanup)
```

This is called the two-phase commit pattern at the application level. Skipping this means seats can get permanently stuck in a held state.

---

### 6. Caching strategy was incomplete

I correctly identified that schedule data (~97 MB) fits in memory, but never said _where_ or _how_:

- **Redis** for schedule and availability (TTL = 5 minutes)
- **Invalidation trigger**: when a booking commits, invalidate the availability key for that `(train_id, date, coach)` combination
- **Write-through** for bookings (write to DB first, then update cache) — never cache-first for transactional data

---

### 7. Schema was never formally written out

Running notes are fine during the interview, but by minute 40 the interviewer should see a clean schema. Formal schema I was converging toward:

```sql
-- Core entities
trains          (train_id, name, train_number, total_coaches)
stations        (station_id, name, city, state)
coaches         (coach_id, train_id, coach_type, total_seats)

-- Route model
routes          (route_id, train_id, station_id, stop_order,
                 scheduled_arrival, scheduled_departure)

-- Availability (the critical table)
seat_inventory  (inventory_id, train_id, journey_date, coach_id,
                 from_stop_order, to_stop_order,
                 seats_confirmed, seats_rac, seats_waitlist)

-- Booking
tickets         (ticket_id, pnr, user_id, train_id, journey_date,
                 src_station_id, dst_station_id, seat_number,
                 coach_type, status, booked_at)

users           (user_id, name, email, phone, created_at)
```

---

## Complete Booking Flow (What It Should Look Like)

```
User searches → GET /trains/search
                 → query routes table for trains running src→dst on date
                 → check seat_inventory for availability
                 → return list with seat counts

User selects  → POST /bookings
                 { train_id, date, src, dst, coach_type, passengers[] }

Server:
  1. Validate WHERE clause exists for the train on given date
  2. BEGIN transaction
  3. SELECT seat_inventory FOR UPDATE (lock segments src_order to dst_order-1)
  4. Check seats_left > 0
     YES → decrement, status = CONFIRMED
     NO  → check RAC quota → RAC
     NO  → assign waitlist position → WAITLIST
  5. Insert ticket with status + generated PNR
  6. COMMIT
  7. Trigger payment initiation (async, outside transaction)
  8. If payment fails within 10 min → cron marks ticket CANCELLED,
     triggers waitlist cascade
```

---

## Key Concepts to Revisit

| Concept                      | Why It Matters Here                                           |
| ---------------------------- | ------------------------------------------------------------- |
| `SELECT FOR UPDATE`          | Row-level locking for concurrent seat booking                 |
| Deadlock prevention          | Always lock rows in consistent order (ascending stop_order)   |
| Two-phase application commit | Handle payment failure without stuck reservations             |
| Cache invalidation           | Availability cache must sync with DB on every booking         |
| PNR generation               | UUID or encoded (train+date+seq) — must be unique and short   |
| Waitlist cascade             | Background job pattern, not inline transaction logic          |
| Stop order vs station ID     | Use integer stop_order for range comparisons, not station IDs |

---

## Self-Assessment

| Area                           | Score      | Note                                               |
| ------------------------------ | ---------- | -------------------------------------------------- |
| Capacity estimation            | 9/10       | Strong, row-level math was correct                 |
| Route model                    | 8/10       | Segment approach was right                         |
| Seat availability model        | 8/10       | The A→E insight was the hardest and I got it       |
| DB choice reasoning            | 7/10       | Right call, could have added indexing strategy     |
| Locking                        | 6/10       | Identified it, but implementation detail was vague |
| API design                     | 2/10       | Almost entirely missing                            |
| Schema formalization           | 4/10       | Converging toward it but never written cleanly     |
| Edge cases (payment, waitlist) | 3/10       | Mentioned at the end, not designed for             |
| **Overall**                    | **6.5/10** | Above average for a fresher                        |

---

## What to Do Before Next System Design Round

- [ ] Practice writing API endpoints first — forces you to think about data contracts
- [ ] Always end with a formal schema on the board, even if rough
- [ ] Learn `SELECT FOR UPDATE` and how DB transactions work at the row level
- [ ] Study the two-phase commit pattern for distributed payment + booking
- [ ] Understand Redis cache invalidation strategies (TTL, write-through, write-behind)
- [ ] Design for failure explicitly: "what if X fails midway?" for every critical step
- [ ] Practice saying: _"I haven't covered X, Y, Z — want me to go there?"_ — self-awareness scores points
