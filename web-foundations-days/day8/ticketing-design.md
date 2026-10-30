# TicketHub --- System Design

## 1. Requirements

TicketHub is an online ticketing platform where users can discover
events, view available seats, reserve seats, pay for tickets, and access
their purchased tickets.

### Functional Requirements

1.  Users can register, log in, and manage their accounts.
2.  Users can browse concerts and events.
3.  Users can view event details, seating arrangements, and available
    seats.
4.  Users can temporarily hold available seats while completing payment.
5.  Users can purchase tickets using an integrated payment gateway.
6.  Users can view their orders and purchased tickets.
7.  The system can release expired seat holds automatically.
8.  Administrators can create events and configure seating arrangements.

### Non-Functional Requirements

-   **Performance:** Event listings and seat availability should load
    quickly, even during popular ticket releases.
-   **Scalability:** The application must support 2 million registered
    users and handle sudden traffic spikes during major concert sales.
-   **Correctness:** A seat must never be sold to two different
    customers. Payment and ticket records must remain consistent.
-   **Fairness:** Customers should be treated fairly during high-demand
    sales. The system should apply consistent rules, prevent automated
    abuse, and avoid allowing one customer to reserve unlimited seats.
-   **Availability:** The application should remain accessible during
    major sales, with graceful handling of overload and temporary
    service failures.
-   **Security:** User accounts, payment information, and personal data
    must be protected using authentication, authorization, HTTPS, and
    secure payment integration.
-   **Reliability:** Failed payments and expired reservations should not
    permanently block seats.

## 2. Capacity Estimates

### Given Information

-   Registered users: 2,000,000
-   Normal daily visitors: 50,000
-   Pages viewed per visitor: 10
-   Normal daily ticket sales: 5,000
-   Popular concert seats: 20,000
-   Customers attempting to purchase during the first 10 minutes:
    200,000
-   Seconds per day: 86,400
-   Seconds in 10 minutes: 600

### A. Normal Traffic

**Daily page views**

50,000 visitors × 10 pages = 500,000 page views per day.

**Average page requests per second**

500,000 ÷ 86,400 = approximately 5.79 requests per second.

Assuming a 5× peak factor for ordinary daily traffic:

5.79 × 5 = approximately 29 requests per second.

**Normal ticket sales**

5,000 tickets per day.

Average tickets sold per second:

5,000 ÷ 86,400 = approximately 0.058 tickets per second.

This is approximately 208 tickets per hour when averaged across the day.

### B. Popular Concert Sale

During a popular concert sale, 200,000 customers attempt to purchase
20,000 seats within 10 minutes.

**Customer arrival rate**

200,000 ÷ 600 = approximately 333 customers per second.

**Available ticket sales rate**

20,000 ÷ 600 = approximately 33.3 successful ticket sales per second if
all seats are sold evenly across the 10-minute period.

**Demand compared to supply**

200,000 customers ÷ 20,000 seats = 10 customers competing for each seat
on average.

Only 10% of the interested customers can obtain a seat.

### C. Traffic Comparison

  ------------------------------------------------------------------------
  Metric                          Normal traffic      Popular concert sale
  -------------------- ------------------------- -------------------------
  Visitors/customers              50,000 per day     200,000 in 10 minutes

  Page views                     500,000 per day       Depends on browsing
                                                                 behaviour

  Average arrival rate      0.58 visitors/second      333 customers/second

  Average page                       5.79/second   Depends on requests per
  requests                                                        customer

  Ticket sales                         5,000/day        Up to 20,000 seats

  Ticket sales rate                 0.058/second               33.3/second
  ------------------------------------------------------------------------

The popular concert sale creates a major traffic spike. Its customer
arrival rate is dramatically higher than normal daily visitor traffic.

The system must therefore support horizontal scaling, caching, request
queues, rate limiting, and safe database transactions.

The 333 customers per second figure represents customer arrivals, not
necessarily total HTTP requests. Each customer may make several requests
while browsing and purchasing.

## 3. API Design

TicketHub exposes a REST API using JSON for communication between the
frontend and backend.

  Method   Endpoint                          Purpose
  -------- --------------------------------- ----------------------------------------
  GET      `/api/events`                     Retrieve available events
  GET      `/api/events/{eventId}`           View event details
  GET      `/api/events/{eventId}/seats`     View seating and availability
  POST     `/api/seat-holds`                 Temporarily reserve available seats
  GET      `/api/seat-holds/{holdId}`        Check the status of a seat hold
  POST     `/api/orders`                     Create an order from a valid seat hold
  POST     `/api/orders/{orderId}/payment`   Initiate payment
  GET      `/api/users/me/tickets`           View purchased tickets

### Example: Retrieve Events

`GET /api/events`

Example response:

``` json
{
  "events": [
    {
      "id": 101,
      "name": "Summer Music Festival",
      "venue": "National Stadium",
      "date": "2027-06-20"
    }
  ]
}
```

### Example: Hold Seats

`POST /api/seat-holds`

Example request:

``` json
{
  "eventId": 101,
  "seatIds": [501, 502]
}
```

Example successful response:

``` json
{
  "holdId": 9001,
  "status": "held",
  "expiresAt": "2027-06-20T10:15:00Z"
}
```

A hold is temporary, for example 10 minutes. If the customer does not
complete payment before expiration, the seats become available again.

If another customer has already held or purchased a requested seat, the
API returns a conflict response such as HTTP 409.

### Example: Initiate Payment

`POST /api/orders/9001/payment`

Example request:

``` json
{
  "paymentMethod": "mobile_money",
  "idempotencyKey": "payment-9001-unique"
}
```

The payment gateway processes the transaction and sends a verified
payment notification to TicketHub. TicketHub marks the order as paid
only after verifying the successful payment result.

### API Error Handling

-   `400 Bad Request`: Invalid input.
-   `401 Unauthorized`: User is not authenticated.
-   `403 Forbidden`: User is not permitted to perform the operation.
-   `404 Not Found`: Event, seat, or order does not exist.
-   `409 Conflict`: Requested seat is already held or sold.
-   `429 Too Many Requests`: Customer has exceeded rate limits.
-   `500 Internal Server Error`: Unexpected server failure.

## 4. Data Model

TicketHub uses a relational database because users, events, seats, and
orders have structured relationships and require transactional
consistency.

### A. Users Table

  Column          Type        Description
  --------------- ----------- ----------------------
  user_id         INTEGER     Primary key
  full_name       VARCHAR     Customer's name
  email           VARCHAR     Unique email address
  password_hash   VARCHAR     Secure password hash
  created_at      TIMESTAMP   Registration date

### B. Events Table

  Column       Type        Description
  ------------ ----------- -------------------
  event_id     INTEGER     Primary key
  event_name   VARCHAR     Name of the event
  venue        VARCHAR     Event location
  event_date   TIMESTAMP   Date and time
  status       VARCHAR     Event status

### C. Seats Table

  -----------------------------------------------------------------------
  Column                  Type                    Description
  ----------------------- ----------------------- -----------------------
  seat_id                 INTEGER                 Primary key

  event_id                INTEGER                 Foreign key to Events

  seat_number             VARCHAR                 Seat identifier

  price                   DECIMAL                 Ticket price

  status                  VARCHAR                 Available, held, or
                                                  sold

  hold_expires_at         TIMESTAMP               Hold expiration time;
                                                  NULL when not held

  held_by_order_id        INTEGER                 Current holding order;
                                                  NULL when not held

  version                 BIGINT                  Optimistic locking
                                                  version, starting at 1
  -----------------------------------------------------------------------

**Example PostgreSQL schema**

``` sql
CREATE TABLE seats (
    seat_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    event_id BIGINT NOT NULL REFERENCES events(event_id),
    seat_number VARCHAR(20) NOT NULL,
    price NUMERIC(10, 2) NOT NULL CHECK (price >= 0),
    status VARCHAR(20) NOT NULL DEFAULT 'available'
        CHECK (status IN ('available', 'held', 'sold')),
    hold_expires_at TIMESTAMPTZ,
    held_by_order_id BIGINT REFERENCES orders(order_id),
    version BIGINT NOT NULL DEFAULT 1 CHECK (version >= 1),
    CONSTRAINT uq_event_seat UNIQUE (event_id, seat_number),
    CONSTRAINT ck_seat_hold_fields CHECK (
        (status = 'held' AND hold_expires_at IS NOT NULL
                         AND held_by_order_id IS NOT NULL)
        OR
        (status <> 'held' AND hold_expires_at IS NULL
                          AND held_by_order_id IS NULL)
    )
);
```

The `version` column supports **optimistic locking**. Whenever a seat is
successfully changed, its version is increased by one. A client submits
the version it last read. If another request has already changed the
seat, the stored version will no longer match and the stale request will
update zero rows.

A unique constraint on `(event_id, seat_number)` prevents duplicate seat
records for the same event.

### D. Orders Table

  Column         Type        Description
  -------------- ----------- --------------------------------------
  order_id       INTEGER     Primary key
  user_id        INTEGER     Foreign key to Users
  event_id       INTEGER     Foreign key to Events
  order_status   VARCHAR     Pending, paid, expired, or cancelled
  total_amount   DECIMAL     Total order amount
  created_at     TIMESTAMP   Order creation time

### E. Order_Items Table

  Column          Type      Description
  --------------- --------- ------------------------
  order_item_id   INTEGER   Primary key
  order_id        INTEGER   Foreign key to Orders
  seat_id         INTEGER   Foreign key to Seats
  ticket_price    DECIMAL   Price at purchase time

A unique constraint on `seat_id` in Order_Items ensures that a seat
cannot be assigned to two different order items.

### Example SQL Transaction: Safely Holding a Seat

The following PostgreSQL example uses an atomic conditional `UPDATE` and
an optimistic version check. The application supplies the seat ID, event
ID, pending order ID, and version it received when the customer viewed
the seat.

``` sql
BEGIN;

UPDATE seats
SET
    status = 'held',
    held_by_order_id = :order_id,
    hold_expires_at = CURRENT_TIMESTAMP + INTERVAL '10 minutes',
    version = version + 1
WHERE
    seat_id = :seat_id
    AND event_id = :event_id
    AND version = :expected_version
    AND (
        status = 'available'
        OR (
            status = 'held'
            AND hold_expires_at <= CURRENT_TIMESTAMP
        )
    )
RETURNING seat_id, status, held_by_order_id,
          hold_expires_at, version;

-- Application decision:
-- If UPDATE returned exactly one row, the hold succeeded.
-- If UPDATE returned no rows, the seat is sold, actively held,
-- does not exist for this event, or the supplied version is stale.
-- In the no-row case, execute ROLLBACK and return HTTP 409 Conflict.

COMMIT;
```

The application must check the `UPDATE ... RETURNING` result before
committing. In production code, it commits only when exactly one row was
returned; otherwise it rolls back. The `:name` values are parameter
placeholders supplied by the backend, not literal PostgreSQL variables.

**Why this is safe when two customers click at the same time:**

1.  Both customers may initially read seat A10 with
    `status = 'available'` and `version = 7`.
2.  The first transaction updates the row. It changes the status to
    `held` and increments the version to 8.
3.  The second transaction cannot successfully apply its stale update.
    PostgreSQL rechecks the update condition after any concurrent row
    update; `version = 7` no longer matches, and the seat is no longer
    available.
4.  The second update returns no row, so the application rolls back and
    tells that customer the seat is no longer available.

This is optimistic locking: requests proceed without taking an explicit
lock at the start, and the version comparison detects whether another
request changed the row first. The conditional update is still atomic at
the database level. All seat-hold and purchase decisions must use the
primary database, not a cached seat status or a read replica.

### Relationships

-   One user can create many orders.
-   One event can have many seats.
-   One event can have many orders.
-   One order can contain multiple seats through Order_Items.
-   Each seat belongs to one event.
-   Each order item refers to one seat.

### Simplified Relationship Diagram

``` text
USERS
  |
  | 1-to-many
  |
ORDERS -------- EVENTS
  |                |
  |                | 1-to-many
  |                |
ORDER_ITEMS      SEATS
  |                |
  +----------------+
        References
        a seat
```

## 5. Preventing Double Booking

Preventing two customers from purchasing the same seat is one of the
most important requirements of TicketHub.

A simple availability check is not enough.

For example, two customers might request seat A10 at nearly the same
time. Both requests could initially see that the seat is available. If
the application does not protect the update, both customers could
receive confirmation for the same seat.

TicketHub prevents this using database transactions, atomic conditional
updates with an optimistic-locking version check, and unique
constraints. The version check rejects stale requests when another
customer changes the seat first.

### Step 1: Customer Requests a Seat Hold

The customer selects a seat and sends a request to:

`POST /api/seat-holds`

The application starts a database transaction and performs an atomic
conditional update against the primary database. It checks the seat
status, hold expiry, and version number in the same SQL statement. The
version number is incremented only when the update succeeds.

### Step 2: Check Availability

The application checks whether the seat is:

-   Available, or
-   Held by another customer with an expired hold.

If the seat is already held by another customer or sold, the request is
rejected with HTTP 409 Conflict.

### Step 3: Create a Temporary Hold

If the seat is available, the application creates a pending order and
updates the seat status to `held`.

It records the order holding the seat and the hold expiration time.

The transaction commits, making the reservation visible to other
requests.

### Step 4: Complete Payment

The customer completes payment through the payment gateway.

After receiving and verifying a successful payment notification, the
application uses another database transaction to:

1.  Verify that the hold belongs to the customer and is still valid.
2.  Mark the order as paid.
3.  Mark the seat as sold.
4.  Create the corresponding ticket/order-item records.
5.  Commit all changes together.

If payment fails or the hold expires, the system releases the seat so
another customer can reserve it.

### Step 5: Database Constraints Provide Additional Protection

The system also uses:

-   A unique constraint on `(event_id, seat_number)` to prevent
    duplicate seat records.
-   A unique constraint on `seat_id` in Order_Items to prevent a seat
    from being assigned to multiple orders.
-   Transactions to ensure related order and seat changes succeed or
    fail together.
-   The `version` column and conditional update to reject stale
    concurrent requests (optimistic locking).
-   Idempotency keys to prevent duplicate payment requests from creating
    duplicate orders or charges.

The application must also handle delayed payment notifications. A
payment received after a hold expires should not automatically sell a
seat that has already been allocated to another customer. The system
needs a payment reconciliation and refund process for that situation.

**Result:** Only one customer can successfully hold and purchase a
particular seat at a time, even when many customers attempt to reserve
it simultaneously.

## 6. System Architecture

### Architecture Diagram

``` text
                    Customers / Browsers
                            |
                       DNS / CDN
                            |
                    Waiting Room / Queue
                            |
                      Load Balancer
                            |
                 +----------+----------+
                 |                     |
             App Server 1          App Server 2
                 |                     |
                 +----------+----------+
                            |
                  +---------+---------+
                  |                   |
              Redis Cache        Payment Gateway
                  |
                  |
             Primary Database
                  |
              Read Replica
                  |
           Background Workers
                  |
             Message Queue
                  |
          Notifications / Cleanup
```

The application also uses the primary database for seat holds, order
creation, and final ticket allocation. These operations must not rely on
cached seat availability.

### Component Explanation

**1. DNS and CDN**

DNS directs users to TicketHub. The CDN delivers static resources such
as images, JavaScript, CSS, and suitable public event information closer
to users.

**2. Waiting Room**

During a popular sale, customers enter a virtual waiting room. It
controls the rate at which customers enter the purchasing system and
helps prevent overwhelming application servers and database connections.

**3. Load Balancer**

Distributes incoming requests across healthy application servers. It
performs health checks and stops sending requests to unhealthy
instances.

**4. Application Servers**

Handle authentication, event browsing, seat holds, order creation, and
payment coordination. Multiple stateless instances can be added as
demand increases.

**5. Redis Cache**

Stores frequently accessed event information and suitable feed or
seating-display data. It reduces repeated database reads. Actual seat
allocation always requires an authoritative database transaction.

**6. Primary Database**

Stores users, events, seats, and orders. It handles seat reservation and
payment-related state changes using transactions and constraints.

**7. Read Replica**

Handles suitable read-heavy queries, such as event browsing. It reduces
the load on the primary database. Seat-hold and purchase operations use
the primary database to avoid relying on potentially stale replica data.

**8. Message Queue and Background Workers**

Handle asynchronous tasks such as sending ticket confirmation emails,
releasing expired holds, and processing notifications. Workers can scale
independently based on queue depth.

**9. Payment Gateway**

Processes customer payments securely. TicketHub verifies payment
notifications before marking orders as paid.

### How the Architecture Survives a Popular Sale

During a popular concert release, 200,000 customers may arrive within 10
minutes.

The waiting room regulates how quickly customers enter the purchasing
system. The load balancer distributes requests across multiple
application servers. Redis and the CDN reduce repeated work for event
browsing.

Seat holds and purchases are handled by the primary database using
transactions and constraints. The message queue moves non-critical tasks
away from the main request path, allowing background workers to process
notifications and expired holds separately.

The application can scale horizontally by adding more application
servers and workers. Monitoring queue depth, database connections,
request latency, and error rates helps identify bottlenecks before they
become major failures.

## 7. Architectural Trade-Offs

### Trade-Off 1: Waiting Room vs. Direct Access

A waiting room controls the number of customers entering the purchasing
system at once. It protects application servers and the database from
sudden traffic spikes.

However, customers may have to wait before they can purchase tickets.
The system must also implement fair admission rules to avoid arbitrary
or unfair access.

**Decision:** TicketHub uses a waiting room during high-demand sales to
protect the purchasing infrastructure while applying consistent customer
admission rules.

### Trade-Off 2: Strong Consistency vs. Read Performance

Using the primary database for every read provides stronger consistency
but increases database load. Read replicas improve read capacity, but
replication lag can cause customers to see outdated information.

**Decision:** TicketHub uses read replicas for suitable browsing
operations but uses the primary database for seat holds and purchases.
This protects correctness where it matters most.

### Trade-Off 3: Synchronous vs. Asynchronous Processing

Synchronous processing provides immediate results but can increase
request latency when performing tasks such as sending emails or
generating tickets.

Asynchronous processing uses a queue and background workers, improving
responsiveness and allowing independent scaling. However, background
tasks may be delayed or fail and therefore require retries and
monitoring.

**Decision:** TicketHub performs seat allocation and order updates
synchronously within database transactions, while notifications and
other non-critical tasks are processed asynchronously.

### Trade-Off 4: Security vs. Convenience During High-Demand Sales

Strict authentication, rate limiting, and bot protection help prevent
abuse and automated ticket hoarding. However, additional checks can
introduce delays and inconvenience legitimate customers.

**Decision:** TicketHub applies rate limits, account verification, and
reasonable purchase limits while keeping the checkout process as simple
as possible.

## 8. Conclusion

TicketHub is designed to support normal event browsing and extremely
high-demand ticket sales. Its architecture combines a waiting room, load
balancing, caching, stateless application servers, a relational
database, and background processing.

The most critical design decision is preventing double booking. Database
transactions, row-level locking, temporary seat holds, and unique
constraints ensure that only one customer can successfully purchase a
particular seat.

The system balances performance, fairness, correctness, scalability, and
reliability. Load testing and continuous monitoring would help validate
its performance as customer demand increases.
