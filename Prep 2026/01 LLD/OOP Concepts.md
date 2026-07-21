The concepts show through in how you design, not in what you name. Focus on applying them naturally:

- **Encapsulation**: Hide state, expose behavior. Make fields private, provide methods for access
- **Abstraction**: Define interfaces for variations. Multiple payment methods? Different vehicle types? Create an interface
- **Polymorphism**: Let objects handle themselves. No type checking, no switch statements on types
- **Inheritance**: Compose behavior, don't inherit it. Reach for interfaces first, use inheritance only when sharing stable implementation

## Encapsulation

If you're designing a class and wondering whether to expose a field or write a getter, write the getter. If you need to return a collection, return an unmodifiable view or a copy.

```python 

class ParkingSpot:
    def occupy(self, vehicle: "Vehicle") -> None:
        ...


class Vehicle:
    def __init__(self, type_: str):
        self.type = type_


class ParkingLot:
    def __init__(self):
        self.spots: list[ParkingSpot] = []  # public, mutable




from typing import Optional

class ParkingSpot:
    def occupy(self, vehicle: "Vehicle") -> None:
        ...


class Vehicle:
    def __init__(self, type_: str):
        self.type = type_


class ParkingLot:
    def __init__(self):
        self._spots: list[ParkingSpot] = []

    def park_vehicle(self, vehicle: Vehicle) -> bool:
        spot = self._find_available_spot(vehicle)
        if spot is None:
            return False
        spot.occupy(vehicle)
        return True

    def _find_available_spot(self, vehicle: Vehicle) -> Optional[ParkingSpot]:
        return self._spots[0] if self._spots else None

    @property
    def spots(self) -> list[ParkingSpot]:
        return list(self._spots)


```

## Abstraction

exposing only what's essential and hiding implementation details behind clear interfaces. You define what something can do without revealing how it does it

The hard part is choosing the right level of abstraction. Too abstract and your interface becomes meaningless (doWork(), handleRequest()). Too specific and you haven't actually abstracted anything. Think about what operations the caller needs to perform, not how those operations happen internally.


```python 

class Order:
    def __init__(self, total: float, credit_card: str):
        self.total = total
        self.credit_card = credit_card


class StripeAPI:
    def set_api_key(self, key: str) -> None:
        ...

    def create_charge(self, amount: float, card: str) -> None:
        ...


class OrderService:
    def __init__(self, api_key: str):
        self.api_key = api_key

    def checkout(self, order: Order) -> None:
        stripe = StripeAPI()
        stripe.set_api_key(self.api_key)
        stripe.create_charge(order.total, order.credit_card)






from abc import ABC, abstractmethod


class PaymentMethod(ABC):
    @abstractmethod
    def process(self, amount: float) -> bool:
        ...


class CreditCardPayment(PaymentMethod):
    def process(self, amount: float) -> bool:
        return True


class PayPalPayment(PaymentMethod):
    def process(self, amount: float) -> bool:
        return True


class Order:
    def __init__(self, total: float, credit_card: str):
        self.total = total
        self.credit_card = credit_card


class OrderService:
    def __init__(self, payment_method: PaymentMethod):
        self.payment_method = payment_method

    def checkout(self, order: Order) -> None:
        self.payment_method.process(order.total)


```

## Polymorphism

Polymorphism is what replaces if (type == "credit") or switch (vehicleType) statements. Instead of checking types, you call the same method and let each object handle itself. Different objects respond to the same action in their own way.

```python 
from enum import Enum
from typing import Optional


class SpotSize(Enum):
    REGULAR = "regular"
    MOTORCYCLE = "motorcycle"
    LARGE = "large"


class ParkingSpot:
    pass


class Vehicle:
    def get_required_spot_size(self) -> SpotSize:
        raise NotImplementedError


class Car(Vehicle):
    def get_required_spot_size(self) -> SpotSize:
        return SpotSize.REGULAR


class Motorcycle(Vehicle):
    def get_required_spot_size(self) -> SpotSize:
        return SpotSize.MOTORCYCLE


class Truck(Vehicle):
    def get_required_spot_size(self) -> SpotSize:
        return SpotSize.LARGE


class ParkingLot:
    def park_vehicle(self, vehicle: Vehicle) -> bool:
        required = vehicle.get_required_spot_size()
        spot = self._find_spot_by_size(required)
        return spot is not None

    def _find_spot_by_size(self, size: SpotSize) -> Optional[ParkingSpot]:
        return None


```

## Inheritance

When a subclass inherits the parent's fields and methods, any change in the parent can break every child. That's the "fragile base class" problem, and it's why inheritance often creates more rigidity than it solves.

A safer alternative is composition + interfaces. An interface defines the behavior, and each class implements it independently. You still get abstraction and polymorphism, but without forcing classes into a parent-child relationship or sharing state they shouldn't.

```python 

from abc import ABC, abstractmethod


class Drivetrain(ABC):
    @abstractmethod
    def start(self) -> None:
        ...


class GasEngine(Drivetrain):
    def start(self) -> None:
        # gas engine startup logic
        ...


class ElectricMotor(Drivetrain):
    def start(self) -> None:
        # electric motor startup logic
        ...


class Car:
    def __init__(self, drivetrain: Drivetrain):
        self.drivetrain = drivetrain

    def start(self) -> None:
        self.drivetrain.start()


```


