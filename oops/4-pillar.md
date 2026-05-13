# The Four Pillars of OOP

### Java · TypeScript · JavaScript — with trade-offs and better answers

---

## 1. Encapsulation

Bundling data and methods into a single unit, and **hiding internal state** from the outside world. The object controls how its data is read or changed.

> **Real-world analogy:** A car — you use the steering wheel and pedals. You never reach into the engine directly.

---

### Java

Java is the strictest enforcer. `private` fields are inaccessible at the bytecode level — no reflection tricks without explicit permission. Getters and setters are idiomatic, which makes intent clear but adds ceremony.

```java
class BankAccount {
    private double balance;   // truly private at bytecode level
    private String owner;

    public BankAccount(String owner, double initial) {
        this.owner = owner;
        this.balance = initial;
    }

    public void deposit(double amount) {
        if (amount > 0) balance += amount;
    }

    public double getBalance() { return balance; }
}
```

---

### TypeScript

TypeScript adds the `private` keyword, but it is **compile-time only** — at runtime it's plain JavaScript. The safer option is the ES2022 `#field` syntax, which is enforced by the JS engine itself.

```typescript
class BankAccount {
  #balance: number; // true runtime privacy (ES2022)
  private owner: string; // compile-time only — stripped at runtime

  constructor(owner: string, initial: number) {
    this.owner = owner;
    this.#balance = initial;
  }

  deposit(amount: number) {
    if (amount > 0) this.#balance += amount;
  }

  get balance() {
    return this.#balance;
  }
}
```

---

### JavaScript

No `private` keyword existed until ES2022's `#fields`. Before that, developers used closures or WeakMaps as workarounds. The closure pattern is arguably more elegant — truly private by scoping, not syntax.

```javascript
// Modern: ES2022 private fields
class BankAccount {
  #balance;

  constructor(owner, initial) {
    this.owner = owner; // public
    this.#balance = initial;
  }

  deposit(amount) {
    if (amount > 0) this.#balance += amount;
  }

  get balance() {
    return this.#balance;
  }
}

// Old pattern: closure-based (truly private, no class needed)
function makeBankAccount(owner, initial) {
  let balance = initial; // private via closure scope
  return {
    deposit: (amt) => {
      balance += amt;
    },
    getBalance: () => balance,
  };
}
```

---

### Language difference

|                     | Java                             | TypeScript           | JavaScript               |
| ------------------- | -------------------------------- | -------------------- | ------------------------ |
| Private enforcement | Runtime (bytecode)               | Compile-time only    | Runtime via `#` (ES2022) |
| Access modifiers    | `private`, `protected`, `public` | Same + `readonly`    | Only `#` (native)        |
| Getters/setters     | Explicit methods                 | `get`/`set` keywords | `get`/`set` keywords     |

---

### Trade-offs

- **Boilerplate creep** — Java's getter/setter pattern adds dozens of lines of code that communicate nothing beyond "this field exists."
- **False security in TypeScript** — `private` in TypeScript is erased at compile time. A JavaScript consumer of your compiled code can freely access "private" members.
- **Verbose for simple data** — For plain data containers (DTOs, config objects), encapsulation adds overhead with little benefit.

---

### Better answers

- **Immutability over encapsulation** — Instead of hiding mutable state, make state immutable. Functional languages like Haskell and Elm do this. In JS/TS, `Object.freeze()` or `readonly` in TypeScript achieves a lighter version.
- **Records (Java 16+)** — Java's `record` keyword auto-generates encapsulated, immutable data classes with zero boilerplate.
- **Plain data + pure functions** — Functional style separates data (plain objects/structs) from behaviour (functions). No encapsulation needed because there's no mutable state to protect.

---

---

## 2. Abstraction

Exposing only **essential features** and hiding complex implementation details. The user of an abstraction doesn't need to know how it works — only what it does.

> **Real-world analogy:** A TV remote — you press Play. You don't know what infrared signals fire, what registers flip, or what motor mechanism spins the disc.

---

### Java

Java provides two tools for abstraction: `abstract class` (partial implementation + contract) and `interface` (pure contract). Java 8+ added `default` methods to interfaces, blurring the line between the two.

```java
// Abstract class: partial implementation
abstract class Shape {
    private String color;

    Shape(String color) { this.color = color; }

    abstract double area();   // subclass MUST implement this

    void describe() {
        System.out.println(color + " shape, area = " + area());
    }
}

// Interface: pure contract
interface Drawable {
    void draw();

    default void highlight() {           // Java 8+ default method
        System.out.println("Highlighted!");
    }
}

class Circle extends Shape implements Drawable {
    private double radius;

    Circle(String color, double r) { super(color); this.radius = r; }

    @Override public double area() { return Math.PI * radius * radius; }
    @Override public void draw() { System.out.println("Drawing circle"); }
}
```

---

### TypeScript

TypeScript has both `abstract class` and `interface`, but both compile away entirely — they exist only for type checking. Interfaces use **structural typing**: any object with matching shape satisfies an interface, no `implements` keyword required.

```typescript
interface Shape {
  area(): number;
  describe(): void;
}

abstract class BaseShape implements Shape {
  constructor(protected color: string) {}

  abstract area(): number; // must override

  describe() {
    console.log(`${this.color} shape, area = ${this.area()}`);
  }
}

class Circle extends BaseShape {
  constructor(
    color: string,
    private radius: number,
  ) {
    super(color);
  }
  area() {
    return Math.PI * this.radius ** 2;
  }
}

// Structural typing: this object satisfies Shape without 'implements'
const square = {
  area: () => 25,
  describe: () => console.log("5×5 square"),
};
```

---

### JavaScript

JavaScript has no interfaces and no `abstract` keyword. Abstraction is enforced by **convention** — throwing errors in base methods that should be overridden.

```javascript
class Shape {
  constructor(color) {
    this.color = color;
  }

  area() {
    // Convention-based abstraction: throw if not overridden
    throw new Error(`${this.constructor.name} must implement area()`);
  }

  describe() {
    console.log(`${this.color} shape, area = ${this.area()}`);
  }
}

class Circle extends Shape {
  constructor(color, radius) {
    super(color);
    this.radius = radius;
  }
  area() {
    return Math.PI * this.radius ** 2;
  }
}
```

---

### Language difference

|                      | Java                                | TypeScript                     | JavaScript                  |
| -------------------- | ----------------------------------- | ------------------------------ | --------------------------- |
| Abstract classes     | Yes (enforced at compile + runtime) | Yes (compile-time only)        | No — convention via `throw` |
| Interfaces           | Yes (enforced)                      | Yes (structural, compile-time) | None                        |
| Multiple abstraction | One class + many interfaces         | One class + many interfaces    | N/A                         |

---

### Trade-offs

- **Leaky abstraction** — Abstractions eventually expose their internals. A `Shape` that has a `setDPI()` method is leaking rendering details into a geometry abstraction.
- **Interface explosion** — In Java enterprise code, you often see `UserService`, `UserServiceImpl`, `IUserService` — three files for one concept.
- **Abstract classes vs interfaces** — Java's choice between `abstract class` and `interface` creates decision fatigue. They overlap significantly since Java 8 added default interface methods.

---

### Better answers

- **Protocols (Swift) / Traits (Rust)** — More composable than abstract classes. A type can conform to many protocols/traits, each adding a slice of behaviour.
- **Algebraic Data Types** — In functional languages, a `Shape` is a union type (`Circle | Rectangle | Triangle`). No class hierarchy needed; pattern matching handles dispatch.
- **Module-level abstraction** — Instead of abstracting via classes, export functions and hide implementation in module scope. The module is the abstraction boundary.

---

---

## 3. Inheritance

A class **acquires properties and behaviour** from a parent class. Promotes code reuse by modelling "is-a" relationships.

> **Real-world analogy:** A `SavingsAccount` IS-A `BankAccount` — it inherits all account features and adds interest calculation on top.

---

### Java

Java supports single class inheritance (`extends`) but multiple interface implementation (`implements`). The `final` keyword prevents subclassing. `super` accesses parent methods and constructors.

```java
class Animal {
    protected String name;

    Animal(String name) { this.name = name; }

    String speak() { return "..."; }

    @Override
    public String toString() { return "Animal(" + name + ")"; }
}

class Dog extends Animal {
    private String breed;

    Dog(String name, String breed) {
        super(name);         // must call parent constructor
        this.breed = breed;
    }

    @Override
    String speak() { return "Woof!"; }

    String fetch() { return name + " fetches the ball!"; }
}

// Animal a = new Dog("Rex", "Labrador");  // polymorphic assignment
// a.speak();  → "Woof!"  (resolved at runtime)
```

---

### TypeScript

Single class inheritance like Java. But TypeScript's structural typing means you often don't need `extends` — satisfying the shape of an interface is enough for compatibility.

```typescript
class Animal {
  constructor(protected name: string) {}

  speak(): string {
    return "...";
  }
}

class Dog extends Animal {
  constructor(
    name: string,
    private breed: string,
  ) {
    super(name);
  }

  speak(): string {
    return "Woof!";
  }
  fetch(): string {
    return `${this.name} fetches!`;
  }
}

// Structural alternative — no extends needed
interface Speaker {
  name: string;
  speak(): string;
}

// Any object with name + speak() satisfies Speaker
const cat: Speaker = { name: "Whiskers", speak: () => "Meow!" };
```

---

### JavaScript

The `class` syntax is **syntactic sugar over prototypal inheritance**. Under the hood, `extends` sets up the prototype chain: `Dog.prototype.__proto__ === Animal.prototype`. Understanding prototypes is essential for debugging.

```javascript
class Animal {
  constructor(name) {
    this.name = name;
  }
  speak() {
    return "...";
  }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name);
    this.breed = breed;
  }
  speak() {
    return "Woof!";
  }
}

// What this actually means under the hood:
// Object.getPrototypeOf(Dog.prototype) === Animal.prototype  → true
// Object.getPrototypeOf(new Dog()) === Dog.prototype          → true

// Pre-ES6 prototypal style (still valid):
function OldAnimal(name) {
  this.name = name;
}
OldAnimal.prototype.speak = function () {
  return "...";
};

function OldDog(name, breed) {
  OldAnimal.call(this, name);
  this.breed = breed;
}
OldDog.prototype = Object.create(OldAnimal.prototype);
OldDog.prototype.speak = function () {
  return "Woof!";
};
```

---

### Language difference

|                     | Java                              | TypeScript                        | JavaScript                         |
| ------------------- | --------------------------------- | --------------------------------- | ---------------------------------- |
| Inheritance type    | Single class, multiple interfaces | Single class, multiple interfaces | Prototypal (class syntax is sugar) |
| `super` keyword     | Yes                               | Yes                               | Yes                                |
| Prevent subclassing | `final class`                     | Not directly                      | Not directly                       |
| Runtime mechanism   | JVM vtable dispatch               | JS prototype chain                | Prototype chain                    |

---

### Trade-offs

- **The fragile base class problem** — A change in `Animal` can break every subclass. The parent and child are tightly coupled across the entire hierarchy.
- **The is-a trap** — A `Duck` IS-A `Bird`, but what if `Bird` has a `fly()` method? Penguins and rubber ducks break the contract. Forced hierarchies violate the Liskov Substitution Principle (LSP).
- **Deep hierarchies** — `Vehicle → MotorVehicle → Car → Sedan → LuxurySedan` becomes impossible to reason about. Which class owns which behaviour?
- **No multiple inheritance** — Java/TS/JS all limit you to one parent class. Diamond inheritance problems are avoided, but so is legitimate shared behaviour.

---

### Better answers

- **Composition over inheritance** — Give a `Duck` a `FlyBehaviour` object instead of inheriting `fly()`. Swap behaviours at runtime. This is the core principle behind the Strategy and Decorator patterns.
- **Mixins (JS/TS)** — Copy methods from multiple sources onto a class without a formal parent. TypeScript mixin patterns give you multiple inheritance safely.
- **Traits (Rust) / Protocols (Swift)** — Behaviour is defined in traits/protocols, not class hierarchies. A `struct` can implement many traits with no inheritance at all.

```typescript
// Composition example — no inheritance
type FlyBehaviour = { fly: () => string };
type QuackBehaviour = { quack: () => string };

function makeDuck(fly: FlyBehaviour, quack: QuackBehaviour) {
  return { ...fly, ...quack, name: "Duck" };
}

const mallard = makeDuck(
  { fly: () => "Flying high!" },
  { quack: () => "Quack!" },
);
// Swap fly behaviour at runtime — impossible with inheritance
```

---

---

## 4. Polymorphism

The same interface or method call **behaves differently** depending on the underlying type at runtime. One call, many forms.

> **Real-world analogy:** Calling `pay(100)` on a PayPal object, a CreditCard object, and a Crypto object does three completely different things — the caller doesn't care which.

---

### Java

Java has two kinds: **compile-time** (method overloading — same name, different parameters) and **runtime** (method overriding — subclass replaces parent method, resolved via dynamic dispatch).

```java
abstract class Payment {
    abstract void pay(double amount);

    // Compile-time polymorphism: overloading
    void pay(double amount, String currency) {
        System.out.println("Paying " + amount + " " + currency);
    }
}

class CreditCard extends Payment {
    @Override
    void pay(double amount) {
        System.out.println("Charging card: $" + amount);
    }
}

class PayPal extends Payment {
    @Override
    void pay(double amount) {
        System.out.println("PayPal transfer: $" + amount);
    }
}

class Crypto extends Payment {
    @Override
    void pay(double amount) {
        System.out.println("Bitcoin: ₿" + (amount / 60000));
    }
}

// Runtime polymorphism — method resolved at runtime, not compile time
List<Payment> methods = List.of(new CreditCard(), new PayPal(), new Crypto());
methods.forEach(p -> p.pay(100.0));
// → Charging card: $100.0
// → PayPal transfer: $100.0
// → Bitcoin: ₿0.00166...
```

---

### TypeScript

TypeScript uses **structural polymorphism** — any object matching the interface shape is accepted, regardless of whether it formally `implements` the interface. No `instanceof` required.

```typescript
interface Payment {
  pay(amount: number): void;
}

class CreditCard implements Payment {
  pay(amount: number) {
    console.log(`Charging card: $${amount}`);
  }
}

class PayPal implements Payment {
  pay(amount: number) {
    console.log(`PayPal: $${amount}`);
  }
}

// Structural polymorphism: any object with pay() works
function checkout(method: Payment, amount: number) {
  method.pay(amount);
}

// This also satisfies Payment without 'implements'
const crypto = {
  pay: (amount: number) => console.log(`₿ ${(amount / 60000).toFixed(5)}`),
};

checkout(new CreditCard(), 100);
checkout(crypto, 100); // works — shape matches
```

---

### JavaScript

Pure **duck typing** — if it has a `pay()` method, it works. There's no formal interface at all. This makes polymorphism effortless but also dangerous — a typo in a method name is a runtime error, not a compile-time one.

```javascript
class CreditCard {
  pay(amount) {
    console.log(`Card: $${amount}`);
  }
}

class PayPal {
  pay(amount) {
    console.log(`PayPal: $${amount}`);
  }
}

// No interface, no contract — just duck typing
function checkout(method, amount) {
  method.pay(amount); // works if method has .pay(), crashes otherwise
}

// Even a plain object works — no class needed
const crypto = {
  pay: (amt) => console.log(`₿ ${(amt / 60000).toFixed(5)}`),
};

checkout(new CreditCard(), 100); // → Card: $100
checkout(new PayPal(), 100); // → PayPal: $100
checkout(crypto, 100); // → ₿ 0.00166

// Danger: this fails at runtime with no warning
checkout({ charge: (amt) => {} }, 100); // TypeError: method.pay is not a function
```

---

### Language difference

|                    | Java                              | TypeScript                              | JavaScript                |
| ------------------ | --------------------------------- | --------------------------------------- | ------------------------- |
| Polymorphism type  | Nominal (explicit type hierarchy) | Structural (shape matching)             | Duck typing (implicit)    |
| Compile-time check | Yes — overloading + overriding    | Yes — structural type check             | No                        |
| Runtime check      | Dynamic dispatch via vtable       | JS prototype dispatch                   | Prototype dispatch        |
| Overloading        | Yes (same name, different params) | Yes (union types / overload signatures) | No (last definition wins) |

---

### Trade-offs

- **Runtime errors in JavaScript** — Duck typing means a missing method crashes at runtime, not at the point where the wrong object was passed.
- **Overloading complexity in Java** — Too many overloaded signatures for one method makes APIs hard to understand (`process(String)`, `process(int)`, `process(String, int)`, ...).
- **Structural typing surprises in TypeScript** — Two completely unrelated classes with the same method signatures are interchangeable. This is powerful but can cause subtle bugs when shapes accidentally match.

---

### Better answers

- **Pattern matching** — In Rust, Haskell, and Scala, you match on the type explicitly with exhaustive checks. The compiler tells you if you forgot a case — something OOP polymorphism cannot do.
- **Discriminated unions (TypeScript)** — A `type Payment = CreditCard | PayPal | Crypto` with a `kind` field and a `switch` statement is often clearer and fully type-safe than an interface hierarchy.
- **Multimethods (Clojure)** — Dispatch on any property of the arguments, not just the first argument's type. Far more flexible than single-dispatch OOP polymorphism.

```typescript
// Discriminated union — exhaustive, type-safe, no class hierarchy
type Payment =
  | { kind: "card"; last4: string }
  | { kind: "paypal"; email: string }
  | { kind: "crypto"; wallet: string };

function pay(method: Payment, amount: number): void {
  switch (method.kind) {
    case "card":
      return console.log(`Card ****${method.last4}: $${amount}`);
    case "paypal":
      return console.log(`PayPal (${method.email}): $${amount}`);
    case "crypto":
      return console.log(`₿ to ${method.wallet}`);
    // TypeScript errors here if you add a new kind and forget to handle it
  }
}
```

---

---

## Summary: Language Philosophy

| Pillar            | Java                                     | TypeScript                               | JavaScript                      |
| ----------------- | ---------------------------------------- | ---------------------------------------- | ------------------------------- |
| **Encapsulation** | Bytecode-level enforcement               | Compile-time (use `#` for runtime)       | `#` fields (ES2022) or closures |
| **Abstraction**   | Abstract classes + interfaces (enforced) | Abstract classes + structural interfaces | Convention (`throw`) only       |
| **Inheritance**   | Single, nominal, JVM vtable              | Single + structural typing               | Prototypal (class = sugar)      |
| **Polymorphism**  | Nominal + overloading                    | Structural + overloads                   | Duck typing                     |

---

## The core OOP trade-off

OOP's deepest problem is **shared mutable state**. Objects hide their data but mutate it on every method call. In concurrent systems this causes race conditions. In large codebases it causes hidden coupling — two objects sharing a reference to the same mutable object silently affect each other.

The second problem is **over-reliance on inheritance** where composition is a better fit. Inheritance couples child to parent permanently; composition lets you swap pieces at runtime.

---

## The better answers

| Problem                             | Modern solution                                                  |
| ----------------------------------- | ---------------------------------------------------------------- |
| Mutable state                       | Immutable data structures (records, `readonly`, `Object.freeze`) |
| Inheritance coupling                | Composition — has-a over is-a                                    |
| No multiple inheritance             | Traits (Rust), Protocols (Swift), Mixins (JS/TS)                 |
| Polymorphism without exhaustiveness | Discriminated unions + pattern matching                          |
| Class boilerplate                   | Functional modules — plain data + pure functions                 |
| All of the above                    | Rust: structs + traits + ownership, no classes at all            |

> **The right answer:** OOP is not wrong — it's a tool. The best codebases mix paradigms: OOP for modelling domain entities, functional style for data transformation, and composition for flexible behaviour.
