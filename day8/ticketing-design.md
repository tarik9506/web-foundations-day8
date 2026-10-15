# TicketHub System Design Document

## 1. Requirements

### Functional Requirements
1. **User Management:** Users can register, log in, and manage their profiles.
2. **Event Discovery:** Users can browse, search, and filter upcoming concerts and events.
3. **Seat Selection:** Users can view real-time seat availability maps for a specific event.
4. **Seat Reservation & Hold:** Users can temporarily hold selected seats for a limited checkout window (e.g., 10 minutes) to prevent others from buying them while paying.
5. **Checkout & Payment:** Users can securely process payments and complete ticket purchases.
6. **Ticket Management:** Users can view their purchased tickets with unique QR codes for venue entry.

### Non-Functional Requirements
* **Correctness (Zero Double-Booking):** Under no circumstances can two users be assigned or sold the same seat.
* **Fairness:** A strict first-come, first-served model enforced via digital waiting rooms and queues during high-demand sales to prevent bot exploitation.
* **Speed & Performance:** Normal API responses must remain under 200ms. During peak sales, page degradation must be graceful, using queuing pages instead of server crashes.
* **Reliability & Availability:** 99.9% uptime, with ACID-compliant database transactions ensuring financial and inventory consistency.

---

## 2. Traffic Estimates

### Baseline Statistics
* **Registered Users:** 2,000,000
* **Normal Day Visitors:** 50,000 daily active users (DAU)
* **Page Views per User:** 10 views/day = **500,000 daily page views**
* **Tickets Sold (Normal):** 5,000 tickets/day

### Normal Traffic Calculations
* **Average Requests Per Second (RPS):** 
  $$\frac{500,000 \text{ views}}{86,400 \text{ seconds}} \approx 5.8 \text{ RPS}$$
* **Peak Normal RPS:** Assuming a 3x multiplier during evening peak hours $\approx 18 \text{–} 20 \text{ RPS}$.

### Big Sale Peak Traffic Calculations
* **Scenario:** A very popular concert goes on sale, drawing **200,000 users** trying to buy **20,000 seats** within the first **10 minutes** (600 seconds).
* **Average Surge RPS (10-minute window):** 
  $$\frac{200,000 \text{ users}}{600 \text{ seconds}} \approx 333 \text{ RPS}$$
* **Instantaneous Spike RPS (Opening Second):** When the sale drops, requests can spike up to **5,000 to 10,000 RPS** as thousands of browsers hit the checkout and seat reservation endpoints simultaneously.

### Comparison
Normal traffic is lightweight and easily handled by a single application server and database instance (~6 RPS). In contrast, the Big Sale surge represents a **1,000x traffic multiplier** during the opening seconds, requiring aggressive caching, a virtual waiting room queue, and horizontal scaling to prevent database lock contention and crashes.

---

## 3. API Design

### 1. Browse Events
* **Endpoint:** `GET /api/events`
* **Query Params:** `category`, `search`, `limit`, `offset`
* **Response:** List of matching events with dates, venues, and starting prices.

### 2. View Seats
* **Endpoint:** `GET /api/events/{eventId}/seats`
* **Response:** Complete seat map layout with current status (`available`, `held`, `booked`).

### 3. Hold Seat(s)
* **Endpoint:** `POST /api/seats/hold`
* **Payload:** `{ "userId": "string", "seatIds": ["uuid1", "uuid2"] }`
* **Response:** Temporary hold token and expiration timestamp (e.g., 10-minute expiry).

### 4. Process Payment & Complete Order
* **Endpoint:** `POST /api/orders`
* **Payload:** `{ "userId": "string", "holdToken": "string", "paymentDetails": { ... } }`
* **Response:** Order confirmation receipt, ticket IDs, and QR code payloads.

### 5. View User Tickets
* **Endpoint:** `GET /api/users/{userId}/tickets`
* **Response:** List of valid tickets owned by the user for upcoming events.

---

## 4. Data Model & Relationships

### Tables

1. **`users`**
   * `id` (UUID, Primary Key)
   * `name` (VARCHAR)
   * `email` (VARCHAR, Unique)
   * `password_hash` (VARCHAR)
   * `created_at` (TIMESTAMP)

2. **`events`**
   * `id` (UUID, Primary Key)
   * `title` (VARCHAR)
   * `description` (TEXT)
   * `date` (TIMESTAMP)
   * `venue` (VARCHAR)
   * `total_seats` (INTEGER)

3. **`seats`**
   * `id` (UUID, Primary Key)
   * `event_id` (UUID, Foreign Key $\rightarrow$ `events.id`)
   * `seat_number` (VARCHAR)
   * `status` (ENUM: `available`, `held`, `booked`)
   * `held_until` (TIMESTAMP, Nullable)

4. **`orders`**
   * `id` (UUID, Primary Key)
   * `user_id` (UUID, Foreign Key $\rightarrow$ `users.id`)
   * `seat_id` (UUID, Foreign Key $\rightarrow$ `seats.id`)
   * `total_amount` (DECIMAL)
   * `status` (ENUM: `pending`, `completed`, `failed`)
   * `created_at` (TIMESTAMP)

### Relationships
* One **Event** has many **Seats** ($1:\text{N}$).
* One **User** has many **Orders** ($1:\text{N}$).
* One **Seat** corresponds to one active **Order** ($1:1$).
* Composite unique constraint on `(event_id, seat_number)` ensures no duplicate seat designations.

---

## 5. Preventing Double-Booking

To guarantee that two users never purchase the same seat, TicketHub utilizes a combination of **database transactions, isolation levels, and row-level locking**:

1. **Atomic Transactions & Pessimistic Locking (`SELECT ... FOR UPDATE`):** When a user requests to hold a seat, the application opens a database transaction with `SERIALIZABLE` or `REPEATABLE READ` isolation. It queries the seat row using an exclusive lock:
   ```sql
   SELECT * FROM seats WHERE id = 'seat_uuid' AND status = 'available' FOR UPDATE;