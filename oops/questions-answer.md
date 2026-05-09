What's the difference between method overloading and method overriding, and how does the JVM resolve each at runtime vs compile time?

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
