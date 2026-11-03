# Ticket Master

![[Excalidraw/TicektMaster.excalidraw.md]]

<!-- EXCALIDRAW_PRACTICE practiceId="cmvdjz24y0a4t01ad7wtvawst" -->

## Requirements
- View event details/seat maps, search events, book specific seats; one ticket per physical seat.
- **Booking:** strong consistency; never double-book.
- **Browsing/search:** high availability, search ~500ms; reads >> writes.
- **Traffic:** extreme bursts when popular events go on sale; waiting room + admission control.
- Practice: 18 Oct 2026. Read-scaling deep dive: up to 10M concurrent readers.

## Core Entities
- **Event:** artistId, venueId, name, description, date.
- **Venue / Artist / User:** seat layout + location / performer details / buyer identity.
- **Ticket:** eventId, seatId, price, status, holdOwner, holdExpiry; unique `(eventId, seatId)`.
- **Booking:** userId, ticketIds, payment reference, status.

## API
```text
GET  /events/:eventId
     -> event + venue + performer + seat availability
GET  /events/search?keyword={q}&start={date}&end={date}&pageSize={n}&page={p}
     -> paginated events
POST /events/:eventId/reservations { ticketIds }
     -> reservationId, expiresAt
POST /reservations/:reservationId/confirm { paymentToken }
     -> bookingId, status
```
- Authenticate the buyer; derive userId server-side. Make confirmation/payment retries idempotent.

## High-Level Design
- **Event reads:** client → API gateway → event service → cache / Postgres; combine event, venue, artist and ticket data in the response.
- **Search:** start with indexed Postgres queries; add Elasticsearch for complex full-text queries or search load.
- **Search indexing:** Postgres → Debezium [[Change Data Capture (CDC)]] → Elasticsearch; search results may lag, booking must revalidate.
- **Booking:** gateway → booking service → authoritative ticket/booking DB + payment gateway.
- **Holds:** ~10-minute checkout window; Redis TTL can support fast hold lookups, but must not be the only double-booking safeguard.

## Deep Dives
### Seat Holds & Payment
- State: `AVAILABLE → HELD → BOOKED`; expired holds become eligible for reservation again.
- Reserve all requested seats atomically in Postgres: lock rows or conditionally update only available/expired tickets; roll back if any seat cannot be held.
- Before payment, atomically verify the reservation owner/state/expiry and extend its deadline; checking then extending separately introduces a race.
- Do not hold a DB transaction open while waiting for payment.
- After verified payment success, atomically validate the live reservation, mark all tickets booked, and create the confirmed booking.
- If the hold expired or seats were reassigned, do not overwrite the new owner; void/refund the payment. A longer TTL alone cannot guarantee correctness.
- Deduplicate payment callbacks/retries; reconcile payments that succeeded while the booking service was unavailable.
- See [[Handling Contention]], [[Concurrency]], [[PostgreSQL]], [[Multi-step Processes]].

### Read & Search Scaling
- Horizontally scale event/search services; cache event metadata using [[Caching|cache-aside]] and use read replicas for browsing.
- Cached/replica seat maps are hints, not booking authority; reservations and confirmation use the primary.
- Keep seat availability fresher than mostly static event/venue metadata.
- Elasticsearch improves full-text/filter queries but adds CDC lag and operational cost.
- See [[Scaling Reads]], [[Prep 2026/03 Engineering Notes/DB Indexing|DB Indexing]], [[Elasticsearch]].
- Alternative browsing model: [[Cassandra#Ticketmaster — Ticket Browsing UI|partition seats by section]]; this does not replace the authoritative booking transaction.

### Live Seat Map
- Booking changes → seat-state events → push layer → SSE/WebSocket clients watching that event.
- Publish committed changes reliably, e.g. transactional outbox + relay; publish hold expiry too.
- Fan out across push servers via a broker such as [[Redis|Redis Pub/Sub]]; each server forwards to its own connections.
- Clients patch changed seats; on reconnect fetch a fresh snapshot because Pub/Sub does not replay missed updates.
- See [[Real-time updates]].

### Popular-Event Waiting Room
- Queue users before checkout; admit only as much traffic as booking capacity can handle.
- Redis sorted set can track queue order; SSE/WebSockets notify admitted users.
- Issue short-lived admission credentials bound to user/session + event; enforce them at reservation/confirmation endpoints.
- Rate-limit joins, deduplicate queue entries, and expire abandoned admissions. A waiting-room UI without backend enforcement is bypassable.

## Revision Notes
- Separate browsing availability from strongly consistent booking.
- Seat updates + booking creation must be one transaction; payment is an external workflow.
- Redis lock expiry + payment success is the key race: DB ownership validation is the final guard.
- Explain the full seat-update path and cross-server fanout, not just “use SSE.”
- Waiting rooms protect booking capacity; they do not replace seat concurrency control.
