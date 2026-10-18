# TicketHub System Design Document

This document outlines the system architecture, API specifications, data model, and high-concurrency scaling strategy for TicketHub, an event ticketing platform.

---

## 1. Requirements

### Functional Requirements

- **Event Browsing & Search:** Users can search and browse events, venues, and available seating sections.
- **Seat Selection & Reservation:** Users can select specific seat numbers and temporarily reserve them for up to 10 minutes during checkout.
- **Payment Processing & Ticket Issuance:** Users can complete transactions securely and receive digital tickets with unique verification codes.
- **Order History:** Users can view purchased ticket details and booking histories.

### Non-Functional Requirements

- **High Availability & Fault Tolerance:** $99.99\%$ availability during normal operations and high-demand ticket drops.
- **Low Read Latency:** Sub-50ms latency for event catalog and seating map queries.
- **Strict Transactional Integrity & Concurrency:** Absolute prevention of double-booking seat reservations under severe concurrent access.
- **Security:** PCI-DSS compliant checkout integrations, rate-limiting, and bot protection.

---

## 2. Traffic & Capacity Estimates

### Baseline Facts

- **Registered Users:** 2,000,000
- **Normal Traffic:** 50,000 daily visitors viewing 10 pages each ($500,000\text{ page views/day}$), selling 5,000 tickets/day.
- **Peak Flash-Sale Traffic:** 200,000 concurrent users attempting to buy 20,000 seats within a 10-minute window ($600\text{ seconds}$).

### Normal Traffic Calculations ($1\text{ day} \approx 100,000\text{ seconds}$)

- **Normal Read Throughput:**
  $$\frac{500,000\text{ page views}}{100,000\text{ seconds}} = \mathbf{5\text{ reads/second}}$$
- **Normal Write Throughput (Ticket Purchases):**
  $$\frac{5,000\text{ sales}}{100,000\text{ seconds}} = \mathbf{0.05\text{ writes/second}}$$

### Peak Flash-Sale Calculations ($10\text{ minutes} = 600\text{ seconds}$)

- **Peak Active Requests:** 200,000 users making ~1 request every 3 seconds = **66,667 requests/second**.
- **Peak Booking Throughput (Write Attempt Rate):**
  $$\frac{200,000\text{ users}}{600\text{ seconds}} \approx \mathbf{333.3\text{ write attempts/second}}$$

---

## 3. REST API Specification

### Endpoints Overview

| Method   | Path                        | Description                                               | Success Status   |
| :------- | :-------------------------- | :-------------------------------------------------------- | :--------------- |
| `GET`    | `/api/v1/events`            | Retrieves list of active events with query filters        | `200 OK`         |
| `GET`    | `/api/v1/events/:id/seats`  | Retrieves real-time seating availability map for an event | `200 OK`         |
| `POST`   | `/api/v1/reservations`      | Holds selected seat(s) for 10 minutes                     | `201 Created`    |
| `POST`   | `/api/v1/payments/checkout` | Processes payment and finalizes ticket issuance           | `200 OK`         |
| `DELETE` | `/api/v1/reservations/:id`  | Cancels or releases a held seat reservation               | `204 No Content` |

### Key API Payload Example (`POST /api/v1/reservations`)

**Request Body:**

```json
{
  "eventId": "evt_99812",
  "seatIds": ["seat_sec101_r5_s12"],
  "userId": "usr_7721"
}
Response Body (201 Created):JSON{
  "reservationId": "res_88123001",
  "status": "HELD",
  "expiresAt": "2026-10-18T10:01:00Z",
  "seatDetails": [
    {
      "seatId": "seat_sec101_r5_s12",
      "price": 120.00
    }
  ]
}
```

## 4. Data Model & Schema

### Entities & Relationships

events $\rightarrow$ seats (One-to-Many): One event contains thousands of individual seat records.users $\rightarrow$ orders (One-to-Many): One user can place multiple ticket orders.orders $\rightarrow$ tickets (One-to-Many): One order can contain multiple purchased tickets.seats $\rightarrow$ tickets / reservations (One-to-One per Event): A single seat for an event maps to at most one confirmed ticket and active reservation.

```SQL
CREATE TABLE events (
    id VARCHAR(64) PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    venue_name VARCHAR(255) NOT NULL,
    event_time TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- 2. Seats Table
CREATE TABLE seats (
    id VARCHAR(64) PRIMARY KEY,
    event_id VARCHAR(64) NOT NULL REFERENCES events(id) ON DELETE CASCADE,
    section VARCHAR(50) NOT NULL,
    row_number VARCHAR(10) NOT NULL,
    seat_number VARCHAR(10) NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'AVAILABLE', -- AVAILABLE, HELD, BOOKED
    version INT NOT NULL DEFAULT 0, -- For optimistic locking
    UNIQUE(event_id, section, row_number, seat_number)
);

-- 3. Users Table
CREATE TABLE users (
    id VARCHAR(64) PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- 4. Orders Table
CREATE TABLE orders (
    id VARCHAR(64) PRIMARY KEY,
    user_id VARCHAR(64) NOT NULL REFERENCES users(id),
    total_amount DECIMAL(10, 2) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'PENDING', -- PENDING, COMPLETED, EXPIRED
    created_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);

-- 5. Tickets Table
CREATE TABLE tickets (
    id VARCHAR(64) PRIMARY KEY,
    order_id VARCHAR(64) NOT NULL REFERENCES orders(id),
    seat_id VARCHAR(64) UNIQUE NOT NULL REFERENCES seats(id),
    qr_code_hash VARCHAR(255) NOT NULL,
    issued_at TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP
);
```

## 5. System Architecture & Double-Booking Prevention

Architecture Diagram

```text

                              +-------------------+
                               |    Client App     |
                               +---------+---------+
                                         |
                                         v
                               +-------------------+
                               | CDN & Cloudflare  |
                               | (Bot Protection)  |
                               +---------+---------+
                                         |
                                         v
                               +-------------------+
                               |   Load Balancer   |
                               +---------+---------+
                                         |
                       +-----------------+-----------------+
                       |                                   |
                       v                                   v
             +-------------------+               +-------------------+
             |    App Server 1   |               |    App Server 2   |
             +----+----+---------+               +----+----+---------+
                  |    |                              |    |
       +----------+    +--------------+    +----------+    +--------------+
       |                              |    |                              |
       v                              v    v                              v
+--------------+              +------------------+             +------------------+
| Redis Queue  |              | PostgreSQL Master|             |  Redis Key-Value |
| (Virtual     |              | (Strict ACIDs)   |             | (Distributed Lock|
| Waiting Room)|              +--------+---------+             |  & TTL Hold)     |
+--------------+                       |                       +------------------+
                                       v
                              +------------------+
                              | PostgreSQL       |
                              | Read Replica     |
                              +------------------+

```

### Component Explanations

**CDN & Cloudflare**: Absorbs static asset requests and enforces rate-limiting / CAPTCHA queues to block malicious bots during flash sales.
**Load Balancer**: Evenly distributes API requests across app servers to prevent server bottlenecks.
**App Servers**: Executes reservation logic, interacts with Redis locks, and handles order processing.
**Redis Virtual Waiting Room**: Queues incoming buyer requests sequentially during heavy sales traffic to prevent server overload.
**Redis Distributed Locks & TTL**: Enforces instant atomic seat locks with a 10-minute expiration timer (TTL).
**PostgreSQL Primary (Master)**: Executes ACID-compliant transaction blocks for final booking writes using optimistic concurrency control.PostgreSQL Read Replica: Serves seat map rendering queries (`GET /events/:id/seats`) without placing read load on the primary writer node.

### How Double-Booking is Prevented (The Core Solution)

To guarantee that two users can never purchase the same seat simultaneously:
**Redis Distributed Mutex Lock:** When a user requests a seat hold, the application executes an atomic Redis command: `SET lock:seat:{seat_id} {user_id} NX EX 600`. The `NX` option ensures the key is set only if it does not already exist. If another user tries simultaneously, Redis rejects it in sub-milliseconds.

**Database Optimistic Concurrency Control:** When updating the SQL database, the query uses version checking:

```SQL
UPDATE seats
SET status = 'HELD', version = version + 1
WHERE id = 'seat_sec101_r5_s12' AND status = 'AVAILABLE' AND version = 0;
```

If another transaction updated the row first, version increments and the affected row count returns 0, immediately aborting the duplicate transaction safely.

## 6. Architectural Trade-offs

**Optimistic Locking vs. Pessimistic Locking**:Trade-off: We selected optimistic locking over database-level row locks (`SELECT ... FOR UPDATE`). Pessimistic database locks cause thread starvation under 200,000 concurrent users. Optimistic locking combined with Redis distributed locks allows high throughput while maintaining absolute transaction isolation.
**Virtual Waiting Room vs. Instant Rejection**:Trade-off: Implementing a virtual waiting room queues users rather than immediately rejecting them with server errors when demand surges. While users wait in a queue for a few seconds, it guarantees predictable system load and a fair purchasing experience.
