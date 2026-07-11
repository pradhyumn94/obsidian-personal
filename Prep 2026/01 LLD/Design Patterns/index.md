
**Creational Patterns**

- **[[Creational - Factory Method|Factory]]** → Use when callers shouldn't care which concrete class gets created.
- **[[Creational - Builder|Builder]]** → Use when an object has lots of optional fields or messy construction details.
- **[[Creational - Singleton|Singleton]]** → Use when you truly need one global instance (rare in interviews).

**Structural Patterns**

- **[[Structural - Decorators|Decorator]]** → Use when you need to layer optional behaviors at runtime without subclass explosion.
- **[[Structural - Facade|Facade]]** → Use when you want to hide internal complexity behind a simple entry point.

**Behavioral Patterns**

- **[[Behavioral - Strategy|Strategy]]** → Use when you're replacing if/else logic with interchangeable behaviors.
- **[[Behavioral - Observer|Observer]]** → Use when multiple components need to react to a single event.
- **[[Behavioral - State machine|State Machine]]** → Use when an object's behavior depends on its current state and transitions get messy.


Creating objects?
├── Hide creation? → Factory
├── Build step-by-step? → Builder
├── Clone? → Prototype
├── Related families? → Abstract Factory
└── Single instance? → Singleton

Adding structure?
├── Add behavior? → Decorator
├── Convert interface? → Adapter
├── Simplify subsystem? → Facade
├── Control access? → Proxy
├── Tree hierarchy? → Composite
└── Separate abstraction & implementation? → Bridge

Changing behavior?
├── One of many algorithms? → Strategy
├── Object changes behavior by state? → State
├── Notify many listeners? → Observer
├── Action as object? → Command
├── Pipeline of handlers? → Chain of Responsibility
├── Central coordinator? → Mediator
├── Fixed algorithm with customizable steps? → Template Method
├── Save/restore state? → Memento
├── Traverse collection? → Iterator
└── Add operations without changing classes? → Visitor
