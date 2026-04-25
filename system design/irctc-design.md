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

TO be continued
