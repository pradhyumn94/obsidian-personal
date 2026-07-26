# Parking Lot - LLD Practice Notes

### Parking Lot LLD Practice - July 26, 2026

#### Key Takeaways

**Requirements**

* When designing a system that issues and processes tickets, always model the full lifecycle of a ticket including invalid states. A ticket can fail validation at exit for two reasons: it does not exist in the system (e.g. fake or unrecognized ID), or it has already been processed (e.g. duplicate exit attempt). Your system should explicitly reject both cases and return a clear error, just like it rejects entry when no spot is available.

* Error handling in low-level design should cover every transition point in your main flows, not just the happy path. For a parking lot, the two main transitions are entry and exit. Each transition has at least one rejection case: entry is rejected when no compatible spot exists, and exit is rejected when the ticket is invalid or already used. Listing these rejection cases explicitly shows the interviewer you are thinking about real-world robustness.

**Entities**

* When you describe types or categories in a design (like vehicle types or spot types), always make them explicit enums rather than implied strings or comments. Enums like VehicleType (MOTORCYCLE, CAR, LARGE) and SpotType (MOTORCYCLE, COMPACT, LARGE) make your design type-safe, self-documenting, and easier to extend without breaking existing code.

* Mark objects as read-only or immutable when their state should not change after creation. A Ticket is a good example because once issued, its spot assignment and entry time should never change. In Java you do this with final fields, in Python with frozen dataclasses. This prevents bugs and signals intent to other developers.

**Class Design**

* Every entity class should be able to answer basic questions about its own state. A ParkingSpot should have an 'occupied' boolean (or equivalent) so it can answer 'am I free?' on its own. Even if ParkingLot also tracks occupancy centrally, the spot itself should not be a dumb data bag that requires an external lookup to determine its own status.

* When you have two valid design approaches (like tracking occupancy on the spot vs. centrally in ParkingLot), you must explicitly commit to one and explain why. In interviews, ambiguity reads as uncertainty. Pick one, state it out loud, and explain the tradeoff so the interviewer knows you are making a deliberate choice.

* Private helper methods like findAvailableSpot(vehicleType) inside ParkingLot keep your public methods readable and isolate complex logic for easier testing. If your enter() method is doing orchestration AND spot-searching AND compatibility-checking, break the inner logic into a private helper. This shows you understand the Single Responsibility Principle at the method level, not just the class level.

**Implementation**

* When mutating multiple pieces of state in a method, compute any values that could throw (like fees) before making changes. This way, if something fails mid-way, you avoid leaving your system in a partially updated state. For example, calculate the fee first, then remove the ticket and free the spot.

* For ID generation in a shared system, use UUIDs instead of sequential counters. Sequential counters require synchronization across concurrent calls and can cause collisions, while UUIDs are statistically unique without needing coordination between threads.

* To implement ceiling division in integer math without using floating point, use this pattern: hours = duration / 60, then if duration % 60 > 0, add 1 to hours. This avoids rounding errors from float conversion and keeps your fee calculation in integer cents.

**Extensibility**

* When you synchronize one mutating method like enter(), always check every other method that touches the same shared state. In a parking lot, exit() removes from the same occupiedSpotIds and activeTickets collections, so it needs the same synchronization treatment. A good habit is to list all shared mutable fields and then verify every method that reads or writes them is protected.

* When a Map lookup could return null (for example, hourlyRates.get(vehicleType) when a new VehicleType is added but the map was not updated), always handle that case explicitly. Either throw a descriptive IllegalArgumentException like 'No rate configured for vehicle type X' or fall back to a default rate. Silently returning null or zero is a hidden bug that is hard to trace in production.

* When you introduce a pluggable strategy in passing (like a SpotAllocationStrategy), briefly state when you would actually apply it. A strong answer sounds like 'I would extract this into a Strategy interface if we need to swap floor-selection logic at runtime, but I would wait for a real requirement before doing so.' This shows you know the pattern AND know when not to use it, which is what senior-level interviews are testing.

​

#### My Answers

**Requirements**

**Q:** (1) Start by asking clarifying questions to understand what you need to build (2) Then list the final set of requirements on the editor to the right.

> *No answer recorded*

**Entities**

**Q:** What are the main entities in the system? A simple list is fine. These will become your classes, enums, etc.

> *No answer recorded*

**Q:** What are the relationships between your entities? Explain how they connect.

> Entity Responsibility ParkingLot:\
> ParkingSpot: Represents one parking space. Has an ID, a type (SpotType), and an occupied flag. Provides methods to check if it's free and to mark it occupied or free. Doesn't know about tickets or pricing, just its own state.\
> Ticket: A record of a parking session. Holds ticket ID, which spot was assigned, vehicle type (VehicleType), and entry time. Read-only after creation. No business logic here, just data that ParkingLot needs to calculate fees and validate exits.\
> VehicleType (enum): Represents the type of vehicle entering the lot — MOTORCYCLE, CAR, LARGE. Used by ParkingLot to match vehicles to compatible spots, and stored on the Ticket for fee calculation.\
> SpotType (enum): Represents the physical size/type of a parking spot — MOTORCYCLE, CAR, LARGE. Used by ParkingSpot to describe its capacity, and by ParkingLot when searching for a compatible available spot. orchestrator. Owns all spots, tracks active tickets, assigns spots at entry, validates tickets and calculates fees at exit.

**Class Design**

**Q:** Define the interface of your classes. For each class, specify its state (properties/fields) and behavior (methods).

> enum VehicleType:\
> enum SpotType:\
> ParkingLot\
> ParkingSpot\
> ParkingSpot(id, spotType)\
> getSpotType() -> SpotType\
> getId() -> String\
> class Ticket: MOTORCYCLE CAR LARGE MOTORCYCLE CAR LARGE spots: List occupiedSpotIds : Set activeTickets: Map<string,Ticket> ParkingLot(spots,hourlyRate) enter(vehicleType) -> Ticket exit(ticketId)-> long id : string type: SpotType id: String spotId: String vehicleType: VehicleType entryTimeMs: long Ticket(id, spotId, vehicleType, entryTime) getId() -> String getSpotId() -> String getVehicleType() -> VehicleType getEntryTime() -> long

**Implementation**

**Q:** Implement the `enter(vehicleType)` method. Walk me through how you find a compatible available spot, create and store the ticket, mark the spot as occupied, and handle the case where no compatible spot exists.

> ParkingLot enter(vehicleType) spot = findAvailableSpot(vehicleType) if spot == null return error occupiedSpotIds.add(spot.id) ticket = createTicket( generateId(), // generate a unique Id spot.id, vehicleType, currentTime() // time in ms ) activeTickets[ticket.id] = ticket return ticket

**Q:** Implement the `exit(ticketId)` method. Show how you validate the ticket, calculate the fee, free the spot, and handle invalid or already-used ticket IDs.

> exit(ticketId): if ticketId == null return error ticket = activeTickets[ticketId] if ticket == null return error exitTime = currentTime() fee = computeFee(ticket.entryTime, exitTime) // this rounds up ms to hour and multiply be hourly rate occupiedSpotIds.remove(ticket.spotId) activeTickets.remove(ticketId) return fee

**Q:** Implement the fee calculation logic — given entry and exit times in milliseconds, return the fee as an integer (in cents). Make sure you handle the ceiling/round-up-to-nearest-hour requirement correctly.

> computeFee(entryTime,exitTime) msDuration = exitTime - entryTime hourDuration = msDuration / (1000*60*60) // Round up to nearest hour (5 minutes becomes 1 hour) if msDuration % (1000 * 60 * 60) > 0 hourDuration++ return hourDuration * hourlyRate

**Extensibility**

**Q:** Right now, `enter()` and `exit()` both read and write `occupiedSpotIds` and `activeTickets`. What happens if two vehicles arrive simultaneously and `findAvailableSpot()` returns the same spot to both threads before either has called `occupiedSpotIds.add()`? How would you fix this in your current design?

> With multiple entrances, we have a race condition where two vehicles could be assigned the same spot. The simplest correct solution is to synchronize the entire enter() method, which serializes all entrance requests. This is sufficient for most parking lots. If we needed higher concurrency, we could use atomic check-and-add on the occupiedSpotIds Set with retry logic. For a parking lot with 3-5 entrances and typical traffic, method-level synchronization is the right choice—it's simple, correct, and performance isn't an issue.

**Q:** Suppose the business wants per-vehicle-type pricing: motorcycles at $3/hr, cars at $5/hr, and large vehicles at $8/hr. Right now `computeFee()` uses a single `hourlyRateCents` field on `ParkingLot` and the `Ticket` already stores `vehicleType`. What's the minimal change that makes this work, and is there a cleaner abstraction you'd introduce instead?

> computeFee(entryTime, exitTime,vehicleType) msDuration = exitTime - entryTime // Guard against clock skew or same-instant exit; treat as 0 billable time if msDuration <= 0 return 0 hourDuration = Math.floor(msDuration / (1000 * 60 * 60)) // Round up to nearest hour (e.g. 5 minutes becomes 1 hour) if msDuration % (1000 * 60 * 60) > 0 hourDuration++ //get correct hourly rate hourlyRate = this.hourlyRates(vehicleType) return hourDuration * hourlyRate

**Q:** Imagine you need to expand the system into a multi-floor garage. You'd want a `ParkingFloor` entity, and potentially different spot-allocation strategies (e.g. fill the lowest floor first vs. distribute evenly across floors). Where would `ParkingFloor` sit relative to your existing `ParkingLot` and `ParkingSpot` classes, and what would you need to change in `findAvailableSpot()` to support pluggable allocation strategies?

> class ParkingLot: - floors: List - activeTickets: Map<String, Ticket>\
> class ParkingFloor: - floorNumber - spots: List + getAvailableSpotCount(spotType) -> int + findAvailableSpot(spotType) -> ParkingSpot\
> findAvailableSpot(vehicleType){ requiredType = mapVehicleTypeToSpotType(vehicleType) for floor in floors spot = floor.findAvailableSpot(requiredType) if spot != null return spot return null }

​

#### Final Code

```typescript
// ═══════════════════════════════════════════════════════════════════════════
// REQUIREMENTS
//
// Example (Tic Tac Toe):
//   1. Two players alternate placing X and O on a 3x3 grid.
//   2. A player wins by completing a row, column, or diagonal.
//   Out of Scope: UI, AI opponent, networking
// ═══════════════════════════════════════════════════════════════════════════

REQUIREMENTS

1. Supports 3 types of vehicles : Mototcycle,car and Large vehicles
2. Lot can have multiple floors
3. If no compatible spot is available, reject entry
4. If ticket is invalid or already used, reject exit
5. Flat hourly based charge 
    - Round up to nearest hour
    - same rate applies to all vehicles regardless of size
6. When a vehicle enters, system automatically assigns an available compatible spot
7. System issues a ticket at entry.
8. When a vehicle exits, user provides ticket ID
   - System validates the ticket
   - Calculates fee based on time spent (hourly, rounded up)
   - Frees the spot for next use

Out of Scope
- Occupancy/capacity display
- Lost ticket handling
- Gate hardware
- payment processing
- pre-booking


// ═══════════════════════════════════════════════════════════════════════════
// ENTITIES & RELATIONSHIPS
//
// Example (Tic Tac Toe):
//   Game, Board, Player
// ═══════════════════════════════════════════════════════════════════════════

Entity	Responsibility
ParkingLot:
 - orchestrator.
 - Owns all spots, tracks active tickets, assigns spots at entry, validates tickets and calculates fees at exit.

ParkingSpot: Represents one parking space. Has an ID, a type (SpotType), and an occupied flag. Provides methods to check if it's free and to mark it occupied or free. Doesn't know about tickets or pricing, just its own state.

Ticket: A record of a parking session. Holds ticket ID, which spot was assigned, vehicle type (VehicleType), and entry time. Read-only after creation. No business logic here, just data that ParkingLot needs to calculate fees and validate exits.

VehicleType (enum): Represents the type of vehicle entering the lot — MOTORCYCLE, CAR, LARGE. Used by ParkingLot to match vehicles to compatible spots, and stored on the Ticket for fee calculation.

SpotType (enum): Represents the physical size/type of a parking spot — MOTORCYCLE, CAR, LARGE. Used by ParkingSpot to describe its capacity, and by ParkingLot when searching for a compatible available spot.


// ═══════════════════════════════════════════════════════════════════════════
// CLASS DESIGN
//
// Example (Tic Tac Toe):
//   class Game:
//     - board: Board
//     - currentPlayer: Player
//     + makeMove(row, col) -> bool
// ═══════════════════════════════════════════════════════════════════════════

enum VehicleType:
- MOTORCYCLE
- CAR
- LARGE

enum SpotType:
- MOTORCYCLE
- CAR
- LARGE

ParkingLot
- spots: List<ParkingSpot>
- occupiedSpotIds : Set<string>  // tracks occupancy centrally; ParkingSpot itself is a stateless data holder
- activeTickets: Map<string,Ticket>
- hourlyRate: long
+ ParkingLot(spots, hourlyRate)
+ enter(vehicleType) -> Ticket
+ exit(ticketId) -> long
- findAvailableSpot(vehicleType) -> ParkingSpot | null  // handles compatibility logic (e.g. MOTORCYCLE fits MOTORCYCLE spot)

ParkingSpot
- id : string
- type: SpotType
// Note: occupancy is tracked centrally in ParkingLot via occupiedSpotIds, not on the spot itself
+ ParkingSpot(id, spotType)
+ getSpotType() -> SpotType
+ getId() -> String

class Ticket:
- id: String
- spotId: String
- vehicleType: VehicleType
- entryTimeMs: long
+ Ticket(id, spotId, vehicleType, entryTime)
+ getId() -> String
+ getSpotId() -> String
+ getVehicleType() -> VehicleType
+ getEntryTime() -> long


// ═══════════════════════════════════════════════════════════════════════════
// IMPLEMENTATION
// ═══════════════════════════════════════════════════════════════════════════

ParkingLot
enter(vehicleType)
    spot = findAvailableSpot(vehicleType)
    if spot == null
        return error

    occupiedSpotIds.add(spot.id)

    // UUID v4 guarantees uniqueness even under concurrent entries
    ticket = createTicket(
        UUID.v4(),     // e.g. "550e8400-e29b-41d4-a716-446655440000"
        spot.id,
        vehicleType,
        currentTime() // time in ms
    )

    activeTickets[ticket.id] = ticket
    return ticket


exit(ticketId):
    if ticketId == null
        return error
    
    ticket = activeTickets[ticketId]
    if ticket == null
        return error

    exitTime = currentTime()

    // Compute fee before mutating state so that if computeFee() throws,
    // the spot and ticket remain intact (no partial state corruption)
    fee = computeFee(ticket.entryTime, exitTime) // rounds up ms to hour and multiplies by hourly rate

    occupiedSpotIds.remove(ticket.spotId)
    activeTickets.remove(ticketId)

    return fee

computeFee(entryTime, exitTime)
    msDuration = exitTime - entryTime

    // Guard against clock skew or same-instant exit; treat as 0 billable time
    if msDuration <= 0
        return 0

    hourDuration = Math.floor(msDuration / (1000 * 60 * 60))

    // Round up to nearest hour (e.g. 5 minutes becomes 1 hour)
    if msDuration % (1000 * 60 * 60) > 0
        hourDuration++

    return hourDuration * hourlyRate


// ═══════════════════════════════════════════════════════════════════════════
// EXTENSIBILITY
// ═══════════════════════════════════════════════════════════════════════════

ParkingLot
- hourlyRates: Map<VehicleType, long> // everything else remains same

computeFee(entryTime, exitTime, vehicleType)
    msDuration = exitTime - entryTime

    // Guard against clock skew or same-instant exit; treat as 0 billable time
    if msDuration <= 0
        return 0
    
    hourDuration = Math.floor(msDuration / (1000 * 60 * 60))

    // Round up to nearest hour (e.g. 5 minutes becomes 1 hour)
    if msDuration % (1000 * 60 * 60) > 0
        hourDuration++

    // Get correct hourly rate; throw if vehicleType has no configured rate
    // (this surfaces misconfiguration early rather than silently charging $0)
    hourlyRate = this.hourlyRates.get(vehicleType)
    if hourlyRate == null
        throw new Error(`No hourly rate configured for vehicle type: ${vehicleType}`)

    return hourDuration * hourlyRate



class ParkingLot:
    - floors: List<ParkingFloor>
    - activeTickets: Map<String, Ticket>

class ParkingFloor:
    - floorNumber
    - spots: List<ParkingSpot>
    + getAvailableSpotCount(spotType) -> int
    + findAvailableSpot(spotType) -> ParkingSpot


findAvailableSpot(vehicleType){
    requiredType = mapVehicleTypeToSpotType(vehicleType)
    for floor in floors
        spot = floor.findAvailableSpot(requiredType)
        if spot != null
            return spot
    return null
}
```

​