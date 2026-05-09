Jvm doesn't just call methods , it has 5 specific opcodes for different kind of calls.

Java Virtual Machine (JVM) bytecode is the instruction set executed by the JVM, serving as an intermediate representation between high-level Java source code and platform-specific machine code.

Opcode -> Used for -> Resolved at
invokestatic -> static methods -> compile time
invokevirtual -> regular instance methods -> Runtime(vtable)
invokeinterface -> interface methods -> Runtime
invokespecial -> constructors, super.method(), pivate method -> Compile time
invokedynamic -> lambdas, dynamic language -> Runtime

invokestatic example:
javaclass MathUtils {
static int square(int x) { return x \* x; }
}

MathUtils.square(5); // → invokestatic

No object involved. The compiler hardcodes the target method directly into the bytecode. Zero lookup needed at runtime.

invokevirtual example:
javaAnimal a = new Dog();
a.speak(); // → invokevirtual

The compiler writes invokevirtual Animal.speak into bytecode — but at runtime the JVM ignores the declared type (Animal) and looks up the actual object's (Dog) vtable to find the real method. This is what enables overriding to work.
