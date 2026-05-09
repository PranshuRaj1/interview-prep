Q1) What's the difference between method overloading and method overriding, and how does the JVM resolve each at runtime vs compile time?

Method Overloading -> Resolved at Compile Time:
Overloading means defining multiple methods with same name but different parameters signatures in the same class.

class Calculator {
int add(int a, int b) { return a + b; }
double add(double a, double b) { return a + b; }
int add(int a, int b, int c) { return a + b + c; }
}

the compiler picks the right method bu inspecting the static(declared) type of arguments at compile time. This is called static dispatch or early binding.

static dispatch is a mechanism where the compiler determines which specific function or method implementation to call at compile time. It offers performance by avoiding runtime lookups, often allowing compiler optimizations like inlining.

early binding also known as static binding or compile-time binding is a process where the compiler directly associates a specific function call or viariable reference with its address during the compilation phase, rather than at runtime.

key rules the compiler uses:

Exact match first
Widening conversions (int -> long -> double)
Autoboxing int -> Integer
Varargs last

Varargs : Varargs (...) lets a method accept any number of arguments. Under the hood, the compiler converts them into an array.

void log(String... messages) { } // varargs

// these all work:
log();
log("hello");
log("a", "b", "c");

Why last in priority? Because varargs is a "catch-all" — if the compiler picked it eagerly, it would ambiguously match almost everything. So the resolution order is:

void print(int x) { System.out.println("int"); }
void print(long x) { System.out.println("long"); }
void print(int... xs) { System.out.println("varargs"); }

print(5); // → "int" (exact match wins, varargs ignored)
print(5L); // → "long" (widening wins, varargs ignored)
print(5, 6); // → "varargs" (nothing else fits two ints)

void log(String... messages) { } // varargs

// these all work:
log();
log("hello");
log("a", "b", "c");

Method Overriding -> Resolved at Runtime

Overriding means a subclass provides a new implementation of a method with the same signature as one in its superclass.

class Animal {
String speak() { return "..."; }
}
class Dog extends Animal {
@Override
String speak() { return "Woof"; }
}

Animal a = new Dog();
a.speak(); // "Woof" — resolved at runtime

Even though a is declared as Animal, the JVM class Dog.speak() because it inspects the actual object type variable. This is dynamic dispatch or late binding , powered by the vtable (cirtual method table).

How the vtable works:
each class gets a vtable -> an array of method pointers.
dog's vtable inherits Animal's entries but overwrites the speak slot with Dog :: speak
at runtime, invokevirtual looks up the vtable of the actual object, not the declared type

Dynamic Dispatch + vtable :
The JVM needs a mechanism to say "ignore what the variable says, check what the object actually is." That mechanism is the vtable.

How the vtable is built — step by step:
Animal class loads → JVM builds Animal's vtable:
┌─────────────────────────────────────┐
│ Animal vtable │
│ slot 0 → Object.toString() │
│ slot 1 → Object.equals() │
│ slot 2 → Object.hashCode() │
│ slot 3 → Animal.speak() ← slot 3 │
└─────────────────────────────────────┘

Dog class loads → JVM builds Dog's vtable:
(copies Animal's table, then overwrites overridden slots)
┌─────────────────────────────────────┐
│ Dog vtable │
│ slot 0 → Object.toString() │
│ slot 1 → Object.equals() │
│ slot 2 → Object.hashCode() │
│ slot 3 → Dog.speak() ← OVERWRITTEN │
└─────────────────────────────────────┘

What happens at a.speak():

1. JVM sees: invokevirtual #speak
2. Looks at the actual object in memory → it's a Dog
3. Goes to Dog's vtable → finds slot 3
4. Slot 3 points to Dog.speak()
5. Executes Dog.speak() → "Woof"

What "late binding" means:
// Early binding (compile time) — target is fixed in bytecode
MathUtils.square(5); // invokestatic → address baked in at compile time

// Late binding (runtime) — target found via vtable at runtime
Animal a = getAnimalFromSomewhere(); // could be Dog, Cat, Bird...
a.speak(); // invokevirtual → JVM decides at runtime which speak() runs

The Full Picture Together
Source code: a.speak()
↓
Compiler: invokevirtual Animal.speak ← only knows declared type
↓
Runtime JVM: 1. What is 'a' actually? → Dog object 2. Dog's vtable slot 3? → Dog.speak() 3. Execute Dog.speak() → "Woof"

One Tricky Scenario — Both at Once
class Animal { String speak() { return "..."; } }
class Dog extends Animal {
@Override String speak() { return "Woof"; } // overrides
String speak(int times) { return "x" + times; } // overloads
}

The Golden Rule
Overloading is about the compiler choosing which method signature to call.
Overriding is about the JVM choosing whose implementation to run.

Q2) Explain the Liskov Substitution Principle with a real example where violating it causes a runtime bug.

In plain terms: a subclass should be fully usable wherever its parent class is expected — no surprises, no broken behavior.

he said : "If S is a subtype of T, then objects of type T may be replaced with objects of type S without altering the correctness of the program."

This is the most famous LSP violation because it feels mathematically correct but breaks code.

class Rectangle {
protected int width;
protected int height;

    public void setWidth(int w)  { this.width = w; }
    public void setHeight(int h) { this.height = h; }

    public int area() { return width * height; }

}

class Square extends Rectangle {

    // A square must keep width == height, so we override both setters
    @Override
    public void setWidth(int w) {
        this.width = w;
        this.height = w;  // force equal
    }

    @Override
    public void setHeight(int h) {
        this.width = h;   // force equal
        this.height = h;
    }

}

void stretchAndPrint(Rectangle r) {
r.setWidth(5);
r.setHeight(10);
// Any rectangle with w=5, h=10 should have area 50
System.out.println("Expected: 50, Got: " + r.area());
}

Rectangle rect = new Rectangle();
stretchAndPrint(rect);

Rectangle sq = new Square();
stretchAndPrint(sq);

behaviour as per contract of rectangle is not working as expected to be. It is not mathmatically wrong

The issue is : Code written for Rectangle no longer behaves as expected when given a Square.

Q3) Why does Java not support multiple inheritance with classes but allows it with interfaces? What problem does this solve, and what problem does it introduce?

Code to explain:
class A {
void hello() { System.out.println("A"); }
}

class B extends A {
@Override
void hello() { System.out.println("B"); }
}

class C extends A {
@Override
void hello() { System.out.println("C"); }
}

class D extends B, C { } // not allowed in Java

D inherits hello() from both B and C.
Which one runs when you call d.hello()?

The JVM has no safe answer. It can't pick B or C arbitrarily that would be silent, unpredictable behaviour. It can't run both that's a different semantic entirely. So Java bans it at the language level rather than guessing.

C++ allows this and forces the programmer to resolve it manually with explicit scope (B::hello()) or virtual inheritance which adds complexity and is a well-known source of bugs.

Why Interfaces Are Allowed for Multiple Inheritance:
interface Flyable {
void fly(); // no body — just a contract
}

interface Swimmable {
void swim(); // no body — just a contract
}

class Duck implements Flyable, Swimmable {
public void fly() { System.out.println("flap flap"); }
public void swim() { System.out.println("splash"); }
}

No diamond problem because there's no inherited implementation to conflict. The class always provides the one concrete body. The interface just enforces the contract.

The Problem Java 8 Introduced — Default Methods

Java 8 added default methods to interfaces so existing interfaces could evolve without breaking all implementing classes

interface Flyable {
default void move() { System.out.println("Flying"); }
}

interface Swimmable {
default void move() { System.out.println("Swimming"); }
}

class Duck implements Flyable, Swimmable {
// Compile error:
// Duck inherits unrelated defaults for move() from Flyable and Swimmable
}

The diamond problem is back — just at the interface level now.

How Java Resolves Default Method Conflicts — 3 Rules
Rule 1 — Class always wins over interface:
interface Flyable {
default void move() { System.out.println("Flying"); }
}

class Animal {
public void move() { System.out.println("Animal moving"); }
}

class Duck extends Animal implements Flyable {
// No conflict — Animal.move() wins, interface default ignored
}

new Duck().move(); // → "Animal moving"

Rule 2 — More specific interface wins:
interface Flyable {
default void move() { System.out.println("Flying"); }
}

interface FastFlyable extends Flyable {
default void move() { System.out.println("Fast Flying"); }
}

class Duck implements Flyable, FastFlyable { }

new Duck().move(); // → "Fast Flying" (FastFlyable is more specific)

Rule 3 — If still ambiguous, class MUST explicitly resolve it:
interface Flyable {
default void move() { System.out.println("Flying"); }
}

interface Swimmable {
default void move() { System.out.println("Swimming"); }
}

class Duck implements Flyable, Swimmable {

    @Override
    public void move() {
        Flyable.super.move();    // explicitly pick one
        // or Swimmable.super.move();
        // or write entirely new logic
    }

}

The compiler forces you to resolve it — it won't guess. This is Java's deliberate design: make ambiguity a compile error, not a runtime surprise.

Q4) What is the diamond problem? How does Java 8+ handle it when two interfaces have default methods with the same signature?

        A        ← defines hello()
       / \
      B   C      ← both override hello()
       \ /
        D        ← inherits from both — which hello() runs?

How Java Resolves Default Method Conflicts — 3 Rules
Rule 1 — Class always wins over interface:
interface Flyable {
default void move() { System.out.println("Flying"); }
}

class Animal {
public void move() { System.out.println("Animal moving"); }
}

class Duck extends Animal implements Flyable {
// No conflict — Animal.move() wins, interface default ignored
}

new Duck().move(); // → "Animal moving"

Rule 2 — More specific interface wins:
interface Flyable {
default void move() { System.out.println("Flying"); }
}

interface FastFlyable extends Flyable {
default void move() { System.out.println("Fast Flying"); }
}

class Duck implements Flyable, FastFlyable { }

new Duck().move(); // → "Fast Flying" (FastFlyable is more specific)

Rule 3 — If still ambiguous, class MUST explicitly resolve it:
interface Flyable {
default void move() { System.out.println("Flying"); }
}

interface Swimmable {
default void move() { System.out.println("Swimming"); }
}

class Duck implements Flyable, Swimmable {

    @Override
    public void move() {
        Flyable.super.move();    // explicitly pick one
        // or Swimmable.super.move();
        // or write entirely new logic
    }

}

Q5) What's the difference between composition and aggregation? When would you choose one over the other in a real system design?

both are "has-a" relationships, but they differ in one critical dimension

Composition — "owns-a" relationship. Child cannot exist without the parent. Parent controls the child's lifecycle.

Aggregation — "has-a" relationship. Child exists independently. Parent just holds a reference

"If I delete the parent, should the child also be deleted?"

YES → Composition
NO → Aggregation

Composition:
class Car {
private final Engine engine;

    Car() {
        this.engine = new Engine(); // Car creates it, Car owns it
    }
    // Engine has no life outside this Car

}

Aggregation:
class Playlist {
private List<Song> songs;

    void add(Song song) { songs.add(song); } // Song passed in — exists outside

}
// Deleting Playlist doesn't delete the Songs

// COMPOSITION — Car creates Engine itself
Car() {
this.engine = new Engine(); // ← new is INSIDE
}

// AGGREGATION — Playlist receives Song from outside
void add(Song song) { // ← object comes IN as parameter
songs.add(song);
}

The new keyword location tells you everything:
new written INSIDE the parent class → Composition (parent owns it)
new written OUTSIDE, passed in → Aggregation (just a reference)

Q6) Can a constructor be private? If yes, what design patterns use this and why?

Yes, absolutely. A constructor can be private. It's a deliberate design decision that means "no outside code can directly instantiate this class."

What Happens Without It
class DatabaseConnection {
DatabaseConnection() { } // public by default
}

// Anyone can do this — uncontrolled
DatabaseConnection c1 = new DatabaseConnection();
DatabaseConnection c2 = new DatabaseConnection();
DatabaseConnection c3 = new DatabaseConnection(); // 100 connections? no one stops you

A private constructor puts you in control of how and how many instances get created.

Design Patterns That Use Private Constructors

1. Singleton Pattern — Exactly One Instance Ever
   The problem it solves: some resources must have exactly one instance — a DB connection pool, a config manager, a logger. Multiple instances would cause conflicts or waste.

class DatabasePool {
private static DatabasePool instance; // the one instance

    private DatabasePool() {               // ← private: no one else can call new
        System.out.println("Pool created");
    }

    public static DatabasePool getInstance() {
        if (instance == null) {
            instance = new DatabasePool(); // only created once
        }
        return instance;
    }

}

DatabasePool p1 = DatabasePool.getInstance();
DatabasePool p2 = DatabasePool.getInstance();

System.out.println(p1 == p2); // → true same object

Why private constructor? Without it, anyone can bypass getInstance() and call new DatabasePool() directly — breaking the "exactly one" guarantee entirely.

Thread-safe version (real production code):
class DatabasePool {
private static volatile DatabasePool instance;

    private DatabasePool() { }

    public static DatabasePool getInstance() {
        if (instance == null) {
            synchronized (DatabasePool.class) {
                if (instance == null) {          // double-checked locking
                    instance = new DatabasePool();
                }
            }
        }
        return instance;
    }

}

2. Factory Method Pattern — Control What Gets Created
   The problem it solves: the caller shouldn't decide which concrete class to instantiate — the factory should, based on input or config.

class Shape {
private String type;

    private Shape(String type) {       // ← private: can't do new Shape() directly
        this.type = type;
    }

    // Factory methods are the only way in
    public static Shape createCircle()    { return new Shape("circle"); }
    public static Shape createSquare()    { return new Shape("square"); }
    public static Shape createTriangle()  { return new Shape("triangle"); }

}
Shape s = Shape.createCircle(); // clean, descriptive
Shape s = new Shape("circle"); // compile error — constructor is private

A private constructor is a contract that says object creation is too important to leave to the caller. It's the foundation of Singleton, Factory, and Builder patterns — each of which centralises creation logic for a different reason: controlling count, controlling type, or controlling validity.
