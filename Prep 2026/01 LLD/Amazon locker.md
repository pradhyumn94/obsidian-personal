# Amazon Locker - LLD Practice Notes

### Amazon Locker LLD Practice - August 23, 2026

#### Key Takeaways

**Requirements**

* When defining system responsibilities in a design, only model what the system itself does, not what humans do afterward. For example, a locker system can identify and unlock expired compartments, but the act of a staff member returning a package to a sender is a human workflow outside the system boundary. Keeping this distinction clean prevents scope creep in your design.

**Entities**

* The orchestrator pattern means one class (like Locker) owns the top-level workflows and coordinates other objects. It knows which Box is available, creates AccessTokens, and handles both deposit and pickup flows. This keeps logic centralized instead of scattered across multiple classes.

* When one object needs to reference another (like AccessToken holding a reference to Box), put that reference directly on the object that needs it. This avoids lookup logic elsewhere and keeps behavior self-contained, which is a key encapsulation win.

* Expiry validation belongs on AccessToken itself, not on Locker or Box. The rule of thumb is to put validation logic on the object that owns the data being validated. This follows the Single Responsibility Principle and makes the token responsible for its own lifecycle.

**Class Design**

* Each entity should own the logic that depends on its own data. For example, an AccessToken class should have its own isExpired() method rather than letting an outside class like Locker calculate expiration. This is called encapsulation and it keeps each class responsible for what it knows about itself.

* When designing a system class like Locker, make sure your interface covers all the core operations a real user or system would need. For a locker system that means deposit, pickup, and cleanup of expired boxes. Thinking through the full lifecycle of an object helps you avoid missing methods.

* Using a map to link access codes to tokens (like accessTokenMapping) is a common and efficient pattern in system design. It gives you O(1) lookup and keeps your retrieval logic simple. When you have entities that need to be found by a key, a map is usually the right data structure to reach for.

**Implementation**

* When you reference a method without parentheses like `box.get_size`, you are pointing at the function object itself, not calling it. Comparing a function object to a value like a Size enum will always be False. Always add parentheses to actually call the method and get its return value, like `box.get_size() == size`.

* When filtering a list of resources to find an available one, you need to check every relevant condition together. For a locker box, that means the size must match AND the box must not already be occupied. Missing either condition means you could hand out a box that is already in use.

**Extensibility**

* When answering a design question verbally, always name the specific class, method, and field involved. Instead of saying 'store expired boxes in a list somewhere', say 'add a pending_staff_removal list on the Locker class, populate it in open_expired_boxes(), and expose a confirm_staff_removal() method that calls _clear_deposit()'. Interviewers evaluate precision, not just direction.

* A two-phase commit pattern for resource reservation works like this: phase 1 is reserveBox() which transitions the box from AVAILABLE to RESERVED and records a timestamp, phase 2 is confirmDeposit() which transitions it to OCCUPIED. This prevents race conditions where two callers grab the same box at the same time.

* When using a RESERVED state, you need a way to release stale reservations or boxes get permanently stuck. Two common approaches are lazy expiry (check the reservedAt timestamp inside reserveBox() before picking a box) and a background job that periodically scans for expired reservations. Pick one and be ready to explain the tradeoff.

* Lock granularity matters for throughput. Locking the entire Locker class blocks all concurrent operations. Instead, lock only the specific read-then-write transition (AVAILABLE to RESERVED) inside _get_available_box(), then lock only the individual box for confirmDeposit() or cancelReservation(). This keeps unrelated operations running in parallel.

​

#### My Answers

**Requirements**

**Q:** (1) Start by asking clarifying questions to understand what you need to build (2) Then list the final set of requirements on the editor to the right.

> *No answer recorded*

**Entities**

**Q:** What are the main entities in the system? A simple list is fine. These will become your classes, enums, etc.

> *No answer recorded*

**Q:** What are the relationships between your entities? Explain how they connect.

> A block that will be orchestrated off the system and will keep a map of boxes to tokens. It will handle pickup and deposit. Then there will be a box. The box will have an ID and a status. It will also keep track of its own state—whether it currently has the package or not. Access control. The access control will hold the forward expiration time and a reference. It will unlock the box. And it will enforce expiry when validated.

**Class Design**

**Q:** Define the interface of your classes. For each class, specify its state (properties/fields) and behavior (methods).

> I’d define a local map that maps access tokens to box IDs and access tokens. Then I can deposit packages into the lockers. The system takes the package size as input and returns an access token when the deposit is successful for the user. If the user then picks up a package, the system takes the access token as input and opens the corresponding box.\
> To handle expiration, staff can open expired boxes. For the checks, I’d remove the expired button. The AccessBox class would include the code expiration information and which box it belongs to. It would implement isExpired, which checks whether the current time is past the expiration time. Then the client box. It would implement marking the size as occupied—markOccupied—and open the box.

**Implementation**

**Q:** Implement `depositPackage(size: Size)` on `Locker`. Walk me through how you find an available `Box`, create the `AccessToken`, update your internal state, and what you return — and what happens if no matching box is available.

> It will deposit the package. First, we check the available boxes for the requested size. Since we subscribe to boxes list this month, we need to put the package into a box from that list. We also check that the box size matches the size we want. If we are not able to find a box of that size, we'll raise an exception: no available box for the requested size. If a box is available, we open the box and mark it as occupied. Then we get an access token, update our map of accessToken to the box, and return that code.

**Q:** Implement `pickup(tokenCode: string)` on `Locker`. What's the full validation flow — including what specific errors you throw for a missing code, an expired token, and a successful pickup — and how do you clean up state after a successful retrieval?

> In check, if an open code is not provided, the open code is provided in the second flow. If no valid open code is provided, the token code is invalid. Get the access token from the token code. The access token was stored earlier using an Apple UCI. If you are not able to find it, throw an exception. Invite access to the concord. Now check whether the access token is expired. An access token will expire. Otherwise, get the compartment—rather, get the box. If you are able to get the box, open it. Then clear deposit. Clearing the deposit means it gets the box, marks the box as free, and invalidates the token.

**Q:** Implement `openExpiredBoxes()` on `Locker`. How do you identify which boxes have expired tokens, and what state do you update — or deliberately leave alone — on both the `Box` and the `accessTokenMapping`?

> For open and open-expired, we’ll iterate over all the access token mappings and check which access tokens have expired. Where those boxes are— from the access token—and open those boxes.

**Extensibility**

**Q:** Right now, `deposit_package()` calls `box.open()` and immediately calls `box.mark_occupied()`, assuming the driver successfully placed the package. What if we wanted to add a physical sensor to each `Box` that confirms a package was actually deposited before generating the access token? Which classes would you need to change, and how would you handle the case where the driver walks away without placing a package after the box opens?

> We will add another flow. Now we’ll have to reserve a box. When the box is reserved, it replaces the package in the box. Then only you will generate the open flow. The flow will change a little. First, you have to develop the box. Then confirm the deposit, using the same reservation ID that you get when reserving the box. If not, cancel the location.

**Q:** Imagine two delivery drivers call `deposit_package(Size.SMALL)` at the exact same time on the same `Locker` instance. Walk me through what could go wrong with your current `_get_available_box()` implementation, and how would you fix it?

> You could get that same box because there’s nobody blocking it. Get the availableBox method. So we can use blocks, but since blocks only need to be on the method, having availableBox works better than using the entire, longer class itself. Otherwise, it will limit the throughput of the locker.

**Q:** Your current `open_expired_boxes()` method iterates over `access_token_mapping` and opens expired boxes, but it never calls `_clear_deposit()` to free those boxes — the `occupied` flag stays `True` and the tokens stay in the map. Was that intentional given the constraint that staff must physically remove packages? And if the business later wanted expired boxes to be automatically freed in the system after staff confirms removal, what's the minimal change to support that without breaking the existing `pickup()` flow?

> And other than just opening the boxes in open expired method, we can keep list of all the boxes that have expired once the staff comes for removal; staff can mark them free.

​

#### Final Code

```python
# ═══════════════════════════════════════════════════════════════════════════
# REQUIREMENTS
#
# Example (Tic Tac Toe):
#   1. Two players alternate placing X and O on a 3x3 grid.
#   2. A player wins by completing a row, column, or diagonal.
#   Out of Scope: UI, AI opponent, networking
# ═══════════════════════════════════════════════════════════════════════════

REQUIREMENTS
1. Carrier deposits a package based on size (small,medium and large)
    - system assigns a matching size box out of available
    - Opens box and returns access token
    - error if no space
2. Upon successful deposit, an access token is generated and returned
   - One access token per package
3. User retrieves package by entering access token
   - System validates code and opens box
   - Throws specific error if code is invalid or expired
4. Access tokens expire after 7 days
   - Expired codes are rejected if used for pickup
   - Package remains in box until staff removes it
5. Staff can trigger cleanup of expired boxes
   - System identifies and unlocks all boxes with expired tokens for staff access
6. Invalid access tokens are rejected with clear error messages
   - Wrong code, already used, or expired - user gets specific feedback

Out of Scope
- Handling the staff removal process itself
- the physical keyboard or input mechanism
- Delivery of the access token to the customer
- UI layer
- multiple locker station
- pricing and payment
- Lockout after failed access token attempts

# ═══════════════════════════════════════════════════════════════════════════
# ENTITIES & RELATIONSHIPS
#
# Example (Tic Tac Toe):
#   Game, Board, Player
# ═══════════════════════════════════════════════════════════════════════════
Locker
- orchestrator
- keeps a map of box:token
- handles pickup and deposit packages

Box
- has ID,size,
- tracks whether a package is physically present in it

AccessToken 
- Represents a bearer token for box access
- holds the code, expiration timestamp, and a reference to the box it unlocks
- enforces expiry when validating.


# ═══════════════════════════════════════════════════════════════════════════
# CLASS DESIGN
#
# Example (Tic Tac Toe):
#   class Game:
#     - board: Board
#     - currentPlayer: Player
#     + makeMove(row, col) -> bool
# ═══════════════════════════════════════════════════════════════════════════
class Locker:
    - Boxes: Box[]
     # string will be box id 
    - accessTokenMapping: Map<string, AccessToken>

    + Locker(Boxes)
    + depositPackage(size:Size) -> string | error
    + pickup(tokenCode:string) -> void | error
    + openExpiredBoxes() -> void

class AccessToken:
    - code: string
    - expiration: timestamp
    - Box: Box

    + AccessToken(code, expiration, Box)
    + isExpired() -> boolean
    + getBox() -> Box
    + getCode() -> string

class Box:
    - size: Size
    - occupied: boolean

    + Box(size)
    + getSize() -> Size
    + isOccupied() -> boolean
    + markOccupied() -> void
    + markFree() -> void
    + open() -> void

enum Size:
    SMALL
    MEDIUM
    LARGE


# ═══════════════════════════════════════════════════════════════════════════
# IMPLEMENTATION
# ═══════════════════════════════════════════════════════════════════════════

class InvalidTokenError(Exception):
    pass


class ExpiredTokenError(Exception):
    pass


class Locker:
    def _get_available_box(self, size: "Size") -> "Box | None":
        for box in self.boxes:
            if box.get_size() == size and not box.is_occupied():
                return box
        return None

    def _generate_access_token(self, box: "Box") -> "AccessToken":
        code = f"{random.randint(0, 999999):06d}"
        expiration = datetime.now() + timedelta(days=7)
        return AccessToken(code, expiration, box)

    def deposit_package(self, size: "Size") -> str:
        box = self._get_available_box(size)
        if box is None:
            raise Exception(f"No available box of size {size}")

        box.open()
        box.mark_occupied()
        access_token = self._generate_access_token(box)
        self.access_token_mapping[access_token.get_code()] = access_token

        return access_token.get_code()

    def _clear_deposit(self, access_token: "AccessToken") -> None:
        box = access_token.get_box()
        box.mark_free()
        self.access_token_mapping.pop(access_token.get_code(), None)

    def pickup(self, tokenCode: str) -> None:
        if not tokenCode:
            raise InvalidTokenError("Invalid token code")

        access_token = self.access_token_mapping.get(tokenCode)
        if access_token is None:
            raise InvalidTokenError("Invalid access token code")

        if access_token.is_expired():
            raise ExpiredTokenError("Access token has expired")

        box = access_token.get_box()
        box.open()
        self._clear_deposit(access_token)


   
    def open_expired_boxes(self) -> None:
        for access_token in self.access_token_mapping.values():
            if access_token.is_expired():
                box = access_token.get_box()
                box.open()


# ═══════════════════════════════════════════════════════════════════════════
# EXTENSIBILITY
# ═══════════════════════════════════════════════════════════════════════════

class Locker:
    def __init__(self):
        self._boxes = []
        self._access_token_mapping = {}  # token_code -> Box
        self._pending_staff_removal: List[str] = []  # token_codes awaiting staff confirmation

    def deposit(self, size: str) -> str:
        box = self._get_available_box(size)
        if box is None:
            raise NoBoxAvailableError(f"No available box for size {size}")
        token_code = self._generate_token(box)
        box.occupied = True
        box.deposited_at = datetime.now()
        self._access_token_mapping[token_code] = box
        box.open()
        return token_code

    def pickup(self, token_code: str) -> None:
        if token_code not in self._access_token_mapping:
            raise InvalidTokenError("Invalid or already-used token")
        box = self._access_token_mapping[token_code]
        box.open()
        self._clear_deposit(token_code)

    def open_expired_boxes(self) -> None:
        """
        Called by a background job or staff-facing tool.
        Opens expired boxes so staff can physically remove packages,
        then records the token in pending_staff_removal.
        Does NOT call _clear_deposit() here — the box stays OCCUPIED
        until staff confirms physical removal via confirm_staff_removal().
        """
        now = datetime.now()
        expired_tokens = [
            token_code
            for token_code, box in self._access_token_mapping.items()
            if (now - box.deposited_at).days > EXPIRY_DAYS
            and token_code not in self._pending_staff_removal
        ]
        for token_code in expired_tokens:
            box = self._access_token_mapping[token_code]
            box.open()
            # Track that this box is awaiting physical removal by staff.
            # _clear_deposit() is intentionally deferred — occupied stays True
            # so the system accurately reflects that the package is still inside.
            self._pending_staff_removal.append(token_code)

    def confirm_staff_removal(self, token_code: str) -> None:
        """
        Called by staff (via UI or API) after physically removing an expired package.
        This is the minimal hook needed to free the box in the system:
        - _clear_deposit() is called unchanged, so pickup() flow is completely untouched.
        - pending_staff_removal is cleaned up.
        """
        if token_code not in self._pending_staff_removal:
            raise InvalidTokenError("Token not pending staff removal")
        self._clear_deposit(token_code)
        self._pending_staff_removal.remove(token_code)

    def _clear_deposit(self, token_code: str) -> None:
        """Frees the box and removes the token. Called by both pickup() and confirm_staff_removal()."""
        box = self._access_token_mapping.pop(token_code)
        box.occupied = False
        box.deposited_at = None

    def _get_available_box(self, size: str):
        for box in self._boxes:
            if box.size == size and not box.occupied:
                return box
        return None

    def _generate_token(self, box) -> str:
        return str(uuid.uuid4())


class Box:
    def __init__(self, size: str):
        self.size = size
        self.occupied = False
        self.deposited_at = None  # set on deposit, cleared on pickup or staff confirmation

    def open(self) -> None:
        # Triggers physical door mechanism
        pass


# Extensibility summary:
# - open_expired_boxes() now populates self._pending_staff_removal (two-line addition)
#   instead of calling _clear_deposit() directly, because the package is still
#   physically inside until staff removes it.
# - confirm_staff_removal(token_code) is the new method on Locker that calls
#   _clear_deposit() unchanged — pickup() is completely untouched.
# - If the business later wants automatic freeing (no staff confirmation required),
#   open_expired_boxes() can call _clear_deposit() directly instead — one-line change,
#   no impact on pickup().

```

​