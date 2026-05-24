# Java — Complete Notes

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Comments](#2-comments)
3. [Data Types](#3-data-types)
4. [Variables](#4-variables)
5. [Operators](#5-operators)
6. [Strings](#6-strings)
7. [Arrays](#7-arrays)
8. [Control Flow](#8-control-flow)
9. [Loops](#9-loops)
10. [Methods](#10-methods)
11. [Classes & Objects](#11-classes--objects)
12. [Access Modifiers](#12-access-modifiers)
13. [Inheritance](#13-inheritance)
14. [Abstract Classes](#14-abstract-classes)
15. [Interfaces](#15-interfaces)
16. [Polymorphism](#16-polymorphism)
17. [Encapsulation](#17-encapsulation)
18. [Enums](#18-enums)
19. [Records (Java 16+)](#19-records-java-16)
20. [Sealed Classes (Java 17+)](#20-sealed-classes-java-17)
21. [Exception Handling](#21-exception-handling)
22. [Collections Framework](#22-collections-framework)
23. [Generics](#23-generics)
24. [Lambda Expressions (Java 8+)](#24-lambda-expressions-java-8)
25. [Streams API (Java 8+)](#25-streams-api-java-8)
26. [Optional (Java 8+)](#26-optional-java-8)
27. [File I/O](#27-file-io)
28. [Concurrency & Multithreading](#28-concurrency--multithreading)
29. [Date & Time (java.time — Java 8+)](#29-date--time-javatime--java-8)
30. [Regular Expressions](#30-regular-expressions)
31. [Annotations](#31-annotations)
32. [Reflection API](#32-reflection-api)
33. [Modules (Java 9+)](#33-modules-java-9)
34. [Input / Output Streams Summary](#34-input--output-streams-summary)
35. [Key Design Patterns](#35-key-design-patterns)
36. [Useful Classes & Utilities](#36-useful-classes--utilities)
37. [JVM & Memory](#37-jvm--memory)
38. [Common Java Commands](#38-common-java-commands)
39. [Quick Reference Cheat Sheet](#39-quick-reference-cheat-sheet)

---

## 1. INTRODUCTION

### What is Java?
- **Object-oriented**, **strongly-typed**, **compiled** programming language.
- Created by **James Gosling** at Sun Microsystems (1995), now owned by Oracle.
- **Write Once, Run Anywhere (WORA)** — compiles to bytecode that runs on JVM.
- Used for enterprise apps, Android, web backends, microservices, big data.

### How Java Works
```
Source Code (.java)  →  Compiler (javac)  →  Bytecode (.class)  →  JVM  →  Machine Code
```

### Key Terms
| Term | Description |
|------|-------------|
| **JDK** | Java Development Kit — compiler + tools + JRE |
| **JRE** | Java Runtime Environment — JVM + libraries |
| **JVM** | Java Virtual Machine — executes bytecode |
| **javac** | Java compiler |
| **java** | Java launcher (runs .class files) |
| **jar** | Java Archive (packaged .class files) |

### Your First Program
```java
// File: HelloWorld.java — filename MUST match class name
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}
```
```bash
javac HelloWorld.java    # compile → HelloWorld.class
java HelloWorld          # run
```

### Java 11+ Single-File Execution
```bash
java HelloWorld.java     # compile + run in one step (single file only)
```

### JShell — Interactive REPL (Java 9+)
```bash
jshell
jshell> System.out.println("Hello")
jshell> int x = 42
jshell> /exit
```

---

## 2. COMMENTS

```java
// Single-line comment

/* Multi-line
   comment */

/**
 * Javadoc comment — generates documentation.
 * @param name the user's name
 * @return greeting string
 * @throws IllegalArgumentException if name is null
 * @since 1.0
 * @author Ritesh
 * @see OtherClass
 * @deprecated Use newMethod() instead
 */
public String greet(String name) {
    return "Hello, " + name;
}
```
```bash
javadoc MyClass.java     # generate HTML docs
```

---

## 3. DATA TYPES

### Primitive Types (8 types)
| Type | Size | Range | Default | Example |
|------|------|-------|---------|---------|
| `byte` | 1 byte | -128 to 127 | 0 | `byte b = 100;` |
| `short` | 2 bytes | -32,768 to 32,767 | 0 | `short s = 30000;` |
| `int` | 4 bytes | -2³¹ to 2³¹-1 | 0 | `int i = 42;` |
| `long` | 8 bytes | -2⁶³ to 2⁶³-1 | 0L | `long l = 100L;` |
| `float` | 4 bytes | ~6-7 decimal digits | 0.0f | `float f = 3.14f;` |
| `double` | 8 bytes | ~15 decimal digits | 0.0d | `double d = 3.14;` |
| `char` | 2 bytes | 0 to 65,535 (Unicode) | '\u0000' | `char c = 'A';` |
| `boolean` | ~1 bit | true / false | false | `boolean b = true;` |

### Literals
```java
// Integer literals
int dec = 42;              // decimal
int hex = 0xFF;            // hexadecimal
int oct = 017;             // octal
int bin = 0b1010;          // binary (Java 7+)
long big = 100_000_000L;   // underscores for readability (Java 7+)

// Float literals
double d = 3.14;
double sci = 2.5e10;
float f = 3.14f;           // f suffix required

// Char literals
char ch = 'A';
char unicode = '\u0041';   // 'A'
char newline = '\n';

// String literal
String s = "Hello";

// Null
String s = null;
```

### Reference Types
- Everything that is not a primitive is a **reference type** (objects).
- Classes, Interfaces, Arrays, Enums, Records, Strings.

### Wrapper Classes (Primitive → Object)
| Primitive | Wrapper |
|-----------|---------|
| `byte` | `Byte` |
| `short` | `Short` |
| `int` | `Integer` |
| `long` | `Long` |
| `float` | `Float` |
| `double` | `Double` |
| `char` | `Character` |
| `boolean` | `Boolean` |

```java
// Autoboxing (primitive → wrapper)
Integer x = 42;              // auto-boxes int to Integer

// Unboxing (wrapper → primitive)
int y = x;                   // auto-unboxes

// Useful methods
Integer.parseInt("42");       // String → int
Integer.valueOf("42");        // String → Integer
Integer.toString(42);         // int → String
Integer.MAX_VALUE;            // 2147483647
Integer.MIN_VALUE;            // -2147483648
Integer.toBinaryString(42);  // "101010"
Integer.toHexString(255);    // "ff"
Integer.toOctalString(8);    // "10"
Double.parseDouble("3.14");
Boolean.parseBoolean("true");
Character.isLetter('A');      // true
Character.isDigit('5');       // true
Character.toUpperCase('a');   // 'A'
```

---

## 4. VARIABLES

### Declaration & Initialization
```java
int age;              // declaration
age = 25;             // assignment
int count = 10;       // declaration + initialization
int a = 1, b = 2;    // multiple

final double PI = 3.14159;   // constant (cannot change)
```

### Variable Types
```java
public class Example {
    // Instance variable — belongs to object
    String name;

    // Static/Class variable — shared by all instances
    static int count = 0;

    // Constant
    static final int MAX = 100;

    public void method() {
        // Local variable — exists in method only
        int x = 10;
    }
}
```

### var — Local Variable Type Inference (Java 10+)
```java
var name = "Ritesh";          // inferred as String
var nums = List.of(1, 2, 3);  // inferred as List<Integer>
var map = new HashMap<String, Integer>();

// Restrictions:
// - Only for local variables with initializer
// - Cannot use for fields, method params, return types
// - Cannot assign null: var x = null;  ← ERROR
```

### Naming Rules
- Can contain letters, digits, `_`, `$`
- Cannot start with digit
- Case-sensitive
- Cannot use reserved keywords
- Convention: `camelCase` for variables/methods, `PascalCase` for classes, `UPPER_SNAKE` for constants

---

## 5. OPERATORS

### Arithmetic
```java
int a = 10, b = 3;
a + b      // 13 — addition
a - b      // 7  — subtraction
a * b      // 30 — multiplication
a / b      // 3  — integer division (both int)
a % b      // 1  — modulus
10.0 / 3   // 3.333... — float division (one operand is double)

a++        // post-increment (use then increment)
++a        // pre-increment (increment then use)
a--        // post-decrement
--a        // pre-decrement
```

### Assignment
```java
x = 10;
x += 5;    // x = x + 5
x -= 5;    // x = x - 5
x *= 5;    // x = x * 5
x /= 5;    // x = x / 5
x %= 5;    // x = x % 5
x &= 5;   // bitwise AND assign
x |= 5;   // bitwise OR assign
x ^= 5;   // bitwise XOR assign
x <<= 2;  // left shift assign
x >>= 2;  // right shift assign
x >>>= 2; // unsigned right shift assign
```

### Comparison
```java
a == b     // equal
a != b     // not equal
a > b      // greater
a < b      // less
a >= b     // greater or equal
a <= b     // less or equal
```

### Logical
```java
a && b     // AND (short-circuit)
a || b     // OR (short-circuit)
!a         // NOT
a & b      // AND (no short-circuit — evaluates both)
a | b      // OR (no short-circuit)
```

### Bitwise
```java
a & b      // AND
a | b      // OR
a ^ b      // XOR
~a         // complement (NOT)
a << 2     // left shift (multiply by 2^n)
a >> 2     // signed right shift (divide by 2^n)
a >>> 2    // unsigned right shift
```

### Ternary
```java
String result = (age >= 18) ? "adult" : "minor";
```

### instanceof
```java
if (obj instanceof String) { ... }

// Pattern matching instanceof (Java 16+)
if (obj instanceof String s) {
    System.out.println(s.length());   // s is already cast
}
```

### Operator Precedence (high → low)
```
()  []  .                      Parentheses, member access
++  --  !  ~  (type)           Unary
*  /  %                        Multiplicative
+  -                           Additive
<<  >>  >>>                    Shift
<  >  <=  >=  instanceof       Relational
==  !=                         Equality
&                              Bitwise AND
^                              Bitwise XOR
|                              Bitwise OR
&&                             Logical AND
||                             Logical OR
?:                             Ternary
=  +=  -=  *=  /=  ...        Assignment
```

---

## 6. STRINGS

### Creating Strings
```java
String s1 = "Hello";                     // string literal (String Pool)
String s2 = new String("Hello");         // new object on heap
String s3 = "Hello" + " World";          // concatenation
String s4 = String.valueOf(42);          // from int
String s5 = """
    Multi-line
    text block
    """;                                  // text block (Java 13+)
```

### String is IMMUTABLE — every modification creates a new String.

### String Methods
```java
String s = "Hello, World!";

// Length
s.length()                    // 13

// Access
s.charAt(0)                   // 'H'
s.indexOf("World")            // 7
s.indexOf("World", 5)         // 7 (search from index 5)
s.lastIndexOf("l")            // 10
s.codePointAt(0)              // 72 (Unicode)

// Substring
s.substring(7)                // "World!"
s.substring(0, 5)             // "Hello"

// Search
s.contains("World")           // true
s.startsWith("Hell")          // true
s.endsWith("!")               // true
s.isEmpty()                   // false
s.isBlank()                   // false (Java 11+, true for whitespace-only)

// Comparison
s.equals("Hello, World!")     // true (content comparison)
s.equalsIgnoreCase("hello, world!")  // true
s.compareTo("Hello")         // positive (lexicographic)
s.compareToIgnoreCase("hello, world!")  // 0

// Case
s.toUpperCase()               // "HELLO, WORLD!"
s.toLowerCase()               // "hello, world!"

// Trim/Strip
"  hello  ".trim()            // "hello" (removes <= U+0020)
"  hello  ".strip()           // "hello" (Unicode-aware, Java 11+)
"  hello  ".stripLeading()    // "hello  "
"  hello  ".stripTrailing()   // "  hello"

// Replace
s.replace('l', 'L')           // "HeLLo, WorLd!"
s.replace("World", "Java")   // "Hello, Java!"
s.replaceAll("[aeiou]", "*")  // regex replace
s.replaceFirst("[aeiou]", "*")  // first match only

// Split
"a,b,c".split(",")           // ["a", "b", "c"]
"a,,b".split(",", -1)        // ["a", "", "b"] (keep empty)
"a,b,c".split(",", 2)        // ["a", "b,c"] (max splits)

// Join
String.join(", ", "a", "b", "c")           // "a, b, c"
String.join("-", List.of("a", "b", "c"))   // "a-b-c"

// Format
String.format("Name: %s, Age: %d", "Ritesh", 25)
String.format("Pi: %.2f", 3.14159)          // "Pi: 3.14"
String.format("%05d", 42)                    // "00042"
String.format("%-10s|", "hi")               // "hi        |"
"Name: %s, Age: %d".formatted("Ritesh", 25) // Java 15+

// Convert
s.toCharArray()               // char[]
s.getBytes()                  // byte[]
s.chars()                     // IntStream
String.valueOf(42)            // "42"
String.valueOf(true)          // "true"

// Repeat (Java 11+)
"ha".repeat(3)                // "hahaha"

// Other
s.intern()                    // add to/get from String Pool
s.hashCode()
s.matches("[A-Z].*")          // regex full-match
"  ".isBlank()                // true (Java 11+)
s.indent(4)                   // indent each line (Java 12+)
s.translateEscapes()          // process \n, \t etc. (Java 15+)
```

### Format Specifiers
| Specifier | Description |
|-----------|-------------|
| `%s` | String |
| `%d` | Integer |
| `%f` | Float |
| `%e` | Scientific notation |
| `%x` / `%X` | Hexadecimal |
| `%o` | Octal |
| `%b` | Boolean |
| `%c` | Character |
| `%n` | Newline (platform-specific) |
| `%10s` | Right-align in 10 chars |
| `%-10s` | Left-align in 10 chars |
| `%05d` | Zero-pad to 5 digits |
| `%.2f` | 2 decimal places |
| `%,d` | Thousands separator |

### StringBuilder (Mutable — for efficient string building)
```java
StringBuilder sb = new StringBuilder();
sb.append("Hello");
sb.append(" ");
sb.append("World");
sb.insert(5, ",");           // "Hello, World"
sb.delete(5, 6);             // "Hello World"
sb.replace(0, 5, "Hi");     // "Hi World"
sb.reverse();                 // "dlroW iH"
sb.length();
sb.charAt(0);
sb.toString();                // convert to String

// StringBuffer — same as StringBuilder but thread-safe (synchronized)
StringBuffer bf = new StringBuffer("Hello");
```

### String Comparison: `==` vs `.equals()`
```java
String a = "Hello";
String b = "Hello";
String c = new String("Hello");

a == b          // true  — same reference (String Pool)
a == c          // false — different objects
a.equals(c)     // true  — same content ← ALWAYS USE THIS
```

---

## 7. ARRAYS

### Declaration & Initialization
```java
// Declare
int[] nums;
String[] names;

// Create with size
int[] nums = new int[5];          // [0, 0, 0, 0, 0]

// Create with values
int[] nums = {1, 2, 3, 4, 5};
int[] nums = new int[]{1, 2, 3};

// Multi-dimensional
int[][] matrix = new int[3][4];
int[][] matrix = {
    {1, 2, 3},
    {4, 5, 6},
    {7, 8, 9}
};

// Jagged array (rows of different lengths)
int[][] jagged = new int[3][];
jagged[0] = new int[]{1, 2};
jagged[1] = new int[]{3, 4, 5};
```

### Access & Modify
```java
int[] nums = {10, 20, 30, 40, 50};
nums[0]            // 10
nums[nums.length - 1]  // 50
nums[2] = 99;      // modify
nums.length        // 5 (property, not method)
```

### Iterate
```java
// for loop
for (int i = 0; i < nums.length; i++) {
    System.out.println(nums[i]);
}

// enhanced for (for-each)
for (int num : nums) {
    System.out.println(num);
}

// 2D array
for (int[] row : matrix) {
    for (int val : row) {
        System.out.print(val + " ");
    }
    System.out.println();
}
```

### java.util.Arrays Utility
```java
import java.util.Arrays;

int[] a = {5, 3, 1, 4, 2};

Arrays.sort(a);                     // [1, 2, 3, 4, 5] — in-place
Arrays.sort(a, 1, 4);              // sort range [1, 4)
Arrays.parallelSort(a);            // parallel sort (large arrays)

int idx = Arrays.binarySearch(a, 3);  // must be sorted first

Arrays.fill(a, 0);                  // fill all with 0
Arrays.fill(a, 1, 3, 99);         // fill range

int[] copy = Arrays.copyOf(a, a.length);       // copy
int[] partial = Arrays.copyOfRange(a, 1, 4);   // copy range

Arrays.equals(a, copy);            // content comparison
Arrays.deepEquals(matrix1, matrix2);  // deep comparison for 2D+

Arrays.toString(a);                 // "[1, 2, 3, 4, 5]"
Arrays.deepToString(matrix);       // "[[1, 2], [3, 4]]"

List<Integer> list = Arrays.asList(1, 2, 3);  // array → list (fixed-size)
int[] arr = {1, 2, 3};
// Arrays.asList(arr) won't work for primitives — use streams:
List<Integer> list = Arrays.stream(arr).boxed().toList();

Arrays.stream(a);                   // IntStream
Arrays.compare(a, b);              // lexicographic compare (Java 9+)
Arrays.mismatch(a, b);             // first differing index (Java 9+)
```

---

## 8. CONTROL FLOW

### if / else if / else
```java
if (age < 13) {
    System.out.println("child");
} else if (age < 18) {
    System.out.println("teen");
} else if (age < 65) {
    System.out.println("adult");
} else {
    System.out.println("senior");
}
```

### switch (Traditional)
```java
switch (day) {
    case 1:
        System.out.println("Monday");
        break;            // break required!
    case 2:
        System.out.println("Tuesday");
        break;
    case 6:
    case 7:               // fall-through for same action
        System.out.println("Weekend");
        break;
    default:
        System.out.println("Other");
}
```

### switch Expression (Java 14+)
```java
// Arrow syntax — no break needed, no fall-through
String result = switch (day) {
    case 1 -> "Monday";
    case 2 -> "Tuesday";
    case 6, 7 -> "Weekend";
    default -> "Other";
};

// With blocks
String result = switch (day) {
    case 1 -> {
        System.out.println("Start of week");
        yield "Monday";     // yield returns value from block
    }
    default -> "Other";
};

// Pattern matching in switch (Java 21+)
String describe(Object obj) {
    return switch (obj) {
        case Integer i -> "Integer: " + i;
        case String s  -> "String: " + s;
        case null      -> "null";
        default        -> "Unknown";
    };
}

// Guarded patterns (Java 21+)
switch (obj) {
    case Integer i when i > 0 -> "positive";
    case Integer i            -> "non-positive";
    default                   -> "not integer";
}
```

---

## 9. LOOPS

### for Loop
```java
for (int i = 0; i < 10; i++) {
    System.out.println(i);
}

// Multiple variables
for (int i = 0, j = 10; i < j; i++, j--) {
    System.out.println(i + " " + j);
}

// Infinite loop
for (;;) { break; }
```

### Enhanced for (for-each)
```java
int[] nums = {1, 2, 3, 4, 5};
for (int num : nums) {
    System.out.println(num);
}

List<String> names = List.of("Alice", "Bob");
for (String name : names) {
    System.out.println(name);
}
```

### while Loop
```java
int i = 0;
while (i < 10) {
    System.out.println(i);
    i++;
}
```

### do-while Loop
```java
// Executes at least once
int i = 0;
do {
    System.out.println(i);
    i++;
} while (i < 10);
```

### Loop Control
```java
// break — exit loop
for (int i = 0; i < 10; i++) {
    if (i == 5) break;
    System.out.println(i);    // 0,1,2,3,4
}

// continue — skip iteration
for (int i = 0; i < 10; i++) {
    if (i % 2 == 0) continue;
    System.out.println(i);    // 1,3,5,7,9
}

// Labeled break/continue (nested loops)
outer:
for (int i = 0; i < 5; i++) {
    for (int j = 0; j < 5; j++) {
        if (j == 3) continue outer;  // skip to next i
        if (i == 4) break outer;     // exit both loops
    }
}
```

---

## 10. METHODS

### Defining Methods
```java
// Syntax: accessModifier returnType methodName(parameters) { body }

public int add(int a, int b) {
    return a + b;
}

public void greet(String name) {
    System.out.println("Hello, " + name);
    // no return needed for void
}

public static void main(String[] args) {
    // static method — belongs to class, not instance
}
```

### Method Overloading (Same name, different parameters)
```java
public int add(int a, int b) { return a + b; }
public double add(double a, double b) { return a + b; }
public int add(int a, int b, int c) { return a + b + c; }
// Determined at COMPILE time based on argument types
```

### Varargs (Variable Arguments)
```java
public int sum(int... numbers) {
    int total = 0;
    for (int n : numbers) total += n;
    return total;
}

sum(1, 2, 3);         // 6
sum(1, 2, 3, 4, 5);   // 15
sum();                 // 0

// Varargs must be last parameter
public void log(String level, String... messages) { }
```

### Pass by Value
```java
// Java ALWAYS passes by value
// Primitives: copy of value
// References: copy of reference (not the object itself)

void modify(int x) { x = 99; }        // original unchanged
void modify(int[] arr) { arr[0] = 99; }  // original IS changed (same object)
void modify(String s) { s = "new"; }   // original unchanged (String immutable)
```

### Recursion
```java
public int factorial(int n) {
    if (n <= 1) return 1;
    return n * factorial(n - 1);
}
```

---

## 11. CLASSES & OBJECTS

### Basic Class
```java
public class Person {
    // Fields (instance variables)
    private String name;
    private int age;

    // Constructor
    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    // No-arg constructor
    public Person() {
        this("Unknown", 0);    // call other constructor
    }

    // Getter
    public String getName() { return name; }

    // Setter
    public void setName(String name) { this.name = name; }

    // Method
    public String greet() {
        return "Hello, I'm " + name;
    }

    // toString
    @Override
    public String toString() {
        return "Person{name='" + name + "', age=" + age + "}";
    }

    // equals and hashCode
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Person person = (Person) o;
        return age == person.age && Objects.equals(name, person.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name, age);
    }
}
```

### Creating Objects
```java
Person p1 = new Person("Ritesh", 25);
Person p2 = new Person();

p1.getName();      // "Ritesh"
p1.greet();        // "Hello, I'm Ritesh"
System.out.println(p1);   // uses toString()
```

### `this` keyword
```java
this.name = name;        // distinguish field from parameter
this.greet();            // call another method
this(name, 0);           // call another constructor (must be first statement)
return this;             // return current object (method chaining)
```

### Static Members
```java
public class Counter {
    private static int count = 0;    // shared by all instances

    public Counter() { count++; }

    public static int getCount() {   // call without instance
        return count;
    }

    // Static block — runs once when class is loaded
    static {
        System.out.println("Class loaded");
    }
}

Counter.getCount();   // called on class, not instance
```

### Initializer Blocks
```java
public class Example {
    private int x;

    // Instance initializer block — runs before each constructor
    {
        x = 10;
        System.out.println("Instance init");
    }

    // Static initializer block — runs once when class loads
    static {
        System.out.println("Static init");
    }
}
// Order: static block → instance block → constructor
```

---

## 12. ACCESS MODIFIERS

| Modifier | Class | Package | Subclass | World |
|----------|-------|---------|----------|-------|
| `public` | ✅ | ✅ | ✅ | ✅ |
| `protected` | ✅ | ✅ | ✅ | ❌ |
| (default/package) | ✅ | ✅ | ❌ | ❌ |
| `private` | ✅ | ❌ | ❌ | ❌ |

### Other Modifiers
| Modifier | Usage |
|----------|-------|
| `static` | Belongs to class, not instance |
| `final` | Cannot be changed/overridden/extended |
| `abstract` | No implementation (must be overridden) |
| `synchronized` | Thread-safe (one thread at a time) |
| `volatile` | Variable always read from main memory |
| `transient` | Excluded from serialization |
| `native` | Implemented in native code (C/C++) |
| `strictfp` | Strict floating-point calculations |
| `sealed` | Restricts which classes can extend (Java 17+) |

---

## 13. INHERITANCE

```java
// Parent class (superclass)
public class Animal {
    protected String name;

    public Animal(String name) {
        this.name = name;
    }

    public void speak() {
        System.out.println(name + " makes a sound");
    }
}

// Child class (subclass) — extends ONE class only
public class Dog extends Animal {
    private String breed;

    public Dog(String name, String breed) {
        super(name);          // call parent constructor (must be first)
        this.breed = breed;
    }

    @Override                  // annotation for overriding
    public void speak() {
        System.out.println(name + " says Woof!");
    }

    public void fetch() {
        System.out.println(name + " fetches ball");
    }
}

// Usage
Animal a = new Dog("Buddy", "Lab");   // polymorphism
a.speak();      // "Buddy says Woof!" — dynamic dispatch
// a.fetch();   // ERROR — Animal reference doesn't know fetch()

// Casting
if (a instanceof Dog d) {    // Java 16+
    d.fetch();                // now OK
}
```

### Rules
- Java supports **single inheritance** only (one parent class)
- `Object` is the root of all classes
- `super.method()` to call parent's version
- `super()` to call parent constructor (must be first line)
- `final` class cannot be extended
- `final` method cannot be overridden

---

## 14. ABSTRACT CLASSES

```java
public abstract class Shape {
    protected String color;

    public Shape(String color) {
        this.color = color;
    }

    // Abstract method — no body, MUST be overridden
    public abstract double area();
    public abstract double perimeter();

    // Concrete method — has implementation
    public String getColor() {
        return color;
    }
}

public class Circle extends Shape {
    private double radius;

    public Circle(String color, double radius) {
        super(color);
        this.radius = radius;
    }

    @Override
    public double area() {
        return Math.PI * radius * radius;
    }

    @Override
    public double perimeter() {
        return 2 * Math.PI * radius;
    }
}

// Shape s = new Shape("red");  // ERROR — cannot instantiate abstract class
Shape s = new Circle("red", 5);
```

---

## 15. INTERFACES

```java
// Interface — defines a contract
public interface Drawable {
    // Constants (implicitly public static final)
    int MAX_SIZE = 100;

    // Abstract methods (implicitly public abstract)
    void draw();
    void resize(int factor);

    // Default method (Java 8+) — has implementation
    default void show() {
        System.out.println("Showing drawable");
    }

    // Static method (Java 8+)
    static Drawable empty() {
        return new Drawable() {
            public void draw() {}
            public void resize(int f) {}
        };
    }

    // Private method (Java 9+) — helper for default methods
    private void log(String msg) {
        System.out.println(msg);
    }
}

// Implement interface (can implement MULTIPLE)
public class Circle implements Drawable, Serializable {
    @Override
    public void draw() {
        System.out.println("Drawing circle");
    }

    @Override
    public void resize(int factor) {
        // ...
    }
}

// Interface extending interface
public interface Clickable extends Drawable {
    void onClick();
}
```

### Interface vs Abstract Class
| Feature | Interface | Abstract Class |
|---------|-----------|---------------|
| Methods | abstract, default, static, private | abstract + concrete |
| Fields | only `public static final` | any |
| Constructor | No | Yes |
| Multiple | class can implement many | class extends one |
| Access | public only (methods) | any access modifier |
| Use when | defining a capability/contract | sharing code among related classes |

### Functional Interface (Java 8+)
```java
@FunctionalInterface
public interface Calculator {
    int calculate(int a, int b);   // exactly ONE abstract method
}

// Use with lambda
Calculator add = (a, b) -> a + b;
Calculator mul = (a, b) -> a * b;
add.calculate(3, 5);   // 8
```

---

## 16. POLYMORPHISM

### Compile-Time (Overloading)
```java
void print(int x) { System.out.println("int: " + x); }
void print(String x) { System.out.println("String: " + x); }
void print(double x) { System.out.println("double: " + x); }
```

### Runtime (Overriding)
```java
Animal animal = new Dog("Buddy");   // upcasting
animal.speak();   // calls Dog's speak() — determined at runtime

// Downcasting
if (animal instanceof Dog dog) {
    dog.fetch();
}
```

### Covariant Return Type
```java
class Animal {
    Animal create() { return new Animal(); }
}
class Dog extends Animal {
    @Override
    Dog create() { return new Dog(); }   // return type can be subtype
}
```

---

## 17. ENCAPSULATION

```java
public class BankAccount {
    private double balance;    // private field

    public BankAccount(double initial) {
        if (initial < 0) throw new IllegalArgumentException("Negative");
        this.balance = initial;
    }

    // Controlled access through methods
    public double getBalance() {
        return balance;
    }

    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Must be positive");
        balance += amount;
    }

    public void withdraw(double amount) {
        if (amount > balance) throw new IllegalArgumentException("Insufficient funds");
        balance -= amount;
    }
}
```

---

## 18. ENUMS

```java
// Basic enum
public enum Day {
    MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY
}

Day today = Day.MONDAY;
Day.values()              // all values as array
Day.valueOf("MONDAY")     // string → enum
today.name()              // "MONDAY"
today.ordinal()           // 0 (position)

// Switch with enum
switch (today) {
    case MONDAY -> System.out.println("Start of week");
    case FRIDAY -> System.out.println("TGIF");
    case SATURDAY, SUNDAY -> System.out.println("Weekend!");
    default -> System.out.println("Midweek");
}

// Enum with fields and methods
public enum Planet {
    MERCURY(3.303e+23, 2.4397e6),
    VENUS(4.869e+24, 6.0518e6),
    EARTH(5.976e+24, 6.37814e6);

    private final double mass;
    private final double radius;

    Planet(double mass, double radius) {
        this.mass = mass;
        this.radius = radius;
    }

    public double getMass() { return mass; }

    public double surfaceGravity() {
        final double G = 6.67300E-11;
        return G * mass / (radius * radius);
    }
}

Planet.EARTH.getMass();
Planet.EARTH.surfaceGravity();

// Enum implementing interface
public enum Operation implements Calculable {
    ADD { public int apply(int a, int b) { return a + b; } },
    SUB { public int apply(int a, int b) { return a - b; } },
    MUL { public int apply(int a, int b) { return a * b; } };

    public abstract int apply(int a, int b);
}
```

---

## 19. RECORDS (Java 16+)

```java
// Immutable data carrier — auto-generates constructor, getters, equals, hashCode, toString
public record Point(double x, double y) { }

Point p = new Point(3.0, 4.0);
p.x()            // 3.0 (accessor, not getX)
p.y()            // 4.0
System.out.println(p);    // Point[x=3.0, y=4.0]

// With validation
public record Person(String name, int age) {
    // Compact constructor
    public Person {
        if (name == null || name.isBlank()) throw new IllegalArgumentException("Name required");
        if (age < 0) throw new IllegalArgumentException("Age must be positive");
        name = name.strip();    // can modify before assignment
    }

    // Additional methods
    public String greeting() {
        return "Hello, " + name;
    }

    // Static factory
    public static Person of(String name) {
        return new Person(name, 0);
    }
}

// Records can implement interfaces but cannot extend classes
public record NamedPoint(String name, double x, double y) implements Serializable { }

// Local records (inside methods)
void process() {
    record Pair(String key, int value) {}
    var p = new Pair("age", 25);
}
```

---

## 20. SEALED CLASSES (Java 17+)

```java
// Restrict which classes can extend/implement
public sealed class Shape permits Circle, Rectangle, Triangle { }

public final class Circle extends Shape { }          // cannot be extended
public sealed class Rectangle extends Shape permits Square { }  // further restricted
public non-sealed class Triangle extends Shape { }   // open for extension

public final class Square extends Rectangle { }

// Sealed interfaces
public sealed interface Expr permits Num, Add, Mul { }
public record Num(int value) implements Expr { }
public record Add(Expr left, Expr right) implements Expr { }
public record Mul(Expr left, Expr right) implements Expr { }

// Great with pattern matching
String describe(Shape s) {
    return switch (s) {
        case Circle c    -> "Circle";
        case Rectangle r -> "Rectangle";
        case Triangle t  -> "Triangle";
        // no default needed — compiler knows all subtypes
    };
}
```

---

## 21. EXCEPTION HANDLING

### try / catch / finally
```java
try {
    int result = 10 / 0;
} catch (ArithmeticException e) {
    System.out.println("Math error: " + e.getMessage());
} catch (NullPointerException | IndexOutOfBoundsException e) {
    // multi-catch (Java 7+)
    System.out.println("Error: " + e.getMessage());
} catch (Exception e) {
    // catch-all (broader catches must come last)
    System.out.println("Unexpected: " + e);
    e.printStackTrace();
} finally {
    // always runs (cleanup)
    System.out.println("Done");
}
```

### Throwing Exceptions
```java
public void setAge(int age) {
    if (age < 0) {
        throw new IllegalArgumentException("Age cannot be negative: " + age);
    }
    this.age = age;
}

// Declare checked exceptions
public void readFile(String path) throws IOException, FileNotFoundException {
    // ...
}
```

### Exception Hierarchy
```
Throwable
├── Error (don't catch — JVM problems)
│   ├── OutOfMemoryError
│   ├── StackOverflowError
│   ├── VirtualMachineError
│   └── ...
└── Exception
    ├── RuntimeException (unchecked — don't need to declare)
    │   ├── NullPointerException
    │   ├── ArrayIndexOutOfBoundsException
    │   ├── ClassCastException
    │   ├── IllegalArgumentException
    │   ├── IllegalStateException
    │   ├── NumberFormatException
    │   ├── ArithmeticException
    │   ├── UnsupportedOperationException
    │   ├── ConcurrentModificationException
    │   └── ...
    ├── IOException (checked)
    │   ├── FileNotFoundException
    │   └── ...
    ├── SQLException (checked)
    ├── ClassNotFoundException (checked)
    ├── InterruptedException (checked)
    └── ...
```

### Checked vs Unchecked
| Checked | Unchecked |
|---------|-----------|
| Must be caught or declared (`throws`) | No requirement |
| Extends `Exception` | Extends `RuntimeException` |
| `IOException`, `SQLException` | `NullPointerException`, `IllegalArgumentException` |
| Compile-time enforcement | Runtime only |

### Custom Exceptions
```java
// Checked exception
public class InsufficientFundsException extends Exception {
    private final double amount;

    public InsufficientFundsException(double amount) {
        super("Insufficient funds: " + amount);
        this.amount = amount;
    }

    public double getAmount() { return amount; }
}

// Unchecked exception
public class InvalidInputException extends RuntimeException {
    public InvalidInputException(String message) {
        super(message);
    }

    public InvalidInputException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

### try-with-resources (Java 7+)
```java
// AutoCloseable resources are automatically closed
try (BufferedReader reader = new BufferedReader(new FileReader("file.txt"));
     BufferedWriter writer = new BufferedWriter(new FileWriter("out.txt"))) {

    String line;
    while ((line = reader.readLine()) != null) {
        writer.write(line);
        writer.newLine();
    }
} catch (IOException e) {
    e.printStackTrace();
}
// reader and writer automatically closed — even if exception occurs

// Effectively final variables (Java 9+)
BufferedReader reader = new BufferedReader(new FileReader("file.txt"));
try (reader) {    // no need to redeclare if effectively final
    // ...
}
```

---

## 22. COLLECTIONS FRAMEWORK

### Collection Hierarchy
```
Iterable
└── Collection
    ├── List (ordered, duplicates allowed)
    │   ├── ArrayList
    │   ├── LinkedList
    │   ├── Vector (legacy, synchronized)
    │   └── Stack (legacy)
    ├── Set (no duplicates)
    │   ├── HashSet
    │   ├── LinkedHashSet (insertion order)
    │   └── TreeSet (sorted)
    └── Queue
        ├── LinkedList
        ├── PriorityQueue
        └── Deque
            ├── ArrayDeque
            └── LinkedList

Map (separate hierarchy — not Collection)
├── HashMap
├── LinkedHashMap (insertion order)
├── TreeMap (sorted by key)
├── Hashtable (legacy, synchronized)
└── ConcurrentHashMap (thread-safe)
```

### List
```java
import java.util.*;

// ArrayList — dynamic array (fast random access)
List<String> list = new ArrayList<>();
list.add("Apple");
list.add("Banana");
list.add(1, "Cherry");        // insert at index
list.get(0);                   // "Apple"
list.set(0, "Avocado");       // replace
list.remove("Banana");         // remove by value
list.remove(0);                // remove by index
list.contains("Cherry");      // true
list.indexOf("Cherry");        // index or -1
list.size();                   // length
list.isEmpty();
list.clear();

list.addAll(List.of("X", "Y", "Z"));
list.subList(1, 3);            // sublist view

// Sort
Collections.sort(list);
list.sort(Comparator.naturalOrder());
list.sort(Comparator.reverseOrder());
list.sort(Comparator.comparing(String::length));

// Immutable lists (Java 9+)
List<String> immutable = List.of("A", "B", "C");
// immutable.add("D");   // UnsupportedOperationException

// Immutable copy (Java 10+)
List<String> copy = List.copyOf(list);

// Convert
String[] arr = list.toArray(new String[0]);
List<String> fromArr = Arrays.asList(arr);       // fixed-size
List<String> fromArr2 = new ArrayList<>(Arrays.asList(arr));  // modifiable

// LinkedList — doubly-linked list (fast insert/delete)
LinkedList<String> linked = new LinkedList<>();
linked.addFirst("First");
linked.addLast("Last");
linked.getFirst();
linked.getLast();
linked.removeFirst();
linked.removeLast();
```

### Set
```java
// HashSet — no order, no duplicates, O(1) operations
Set<String> set = new HashSet<>();
set.add("Apple");
set.add("Banana");
set.add("Apple");     // ignored — duplicate
set.contains("Apple");  // true
set.remove("Apple");
set.size();           // unique count

// LinkedHashSet — maintains insertion order
Set<String> ordered = new LinkedHashSet<>();

// TreeSet — sorted order (natural or comparator)
Set<String> sorted = new TreeSet<>();
sorted.add("Banana");
sorted.add("Apple");
sorted.add("Cherry");
// iteration: Apple, Banana, Cherry

TreeSet<Integer> nums = new TreeSet<>();
nums.first();          // smallest
nums.last();           // largest
nums.headSet(5);       // elements < 5
nums.tailSet(5);       // elements >= 5
nums.subSet(2, 8);     // range [2, 8)
nums.floor(5);         // greatest <= 5
nums.ceiling(5);       // smallest >= 5

// Set operations
Set<Integer> a = new HashSet<>(List.of(1, 2, 3, 4));
Set<Integer> b = new HashSet<>(List.of(3, 4, 5, 6));

Set<Integer> union = new HashSet<>(a);
union.addAll(b);                  // {1,2,3,4,5,6}

Set<Integer> intersection = new HashSet<>(a);
intersection.retainAll(b);        // {3,4}

Set<Integer> difference = new HashSet<>(a);
difference.removeAll(b);          // {1,2}

// Immutable (Java 9+)
Set<String> immutable = Set.of("A", "B", "C");
```

### Map
```java
// HashMap — key-value pairs, O(1) operations
Map<String, Integer> map = new HashMap<>();
map.put("Alice", 90);
map.put("Bob", 85);
map.put("Alice", 95);    // overwrites previous value

map.get("Alice");          // 95
map.getOrDefault("Eve", 0);  // 0 (default)
map.containsKey("Bob");   // true
map.containsValue(85);    // true
map.remove("Bob");
map.size();
map.isEmpty();
map.clear();

// Iterate
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    System.out.println(entry.getKey() + ": " + entry.getValue());
}
for (String key : map.keySet()) { ... }
for (Integer val : map.values()) { ... }
map.forEach((k, v) -> System.out.println(k + ": " + v));

// Useful methods (Java 8+)
map.putIfAbsent("Eve", 70);
map.computeIfAbsent("key", k -> expensiveCompute(k));
map.computeIfPresent("Alice", (k, v) -> v + 10);
map.compute("Alice", (k, v) -> v == null ? 0 : v + 1);
map.merge("Alice", 5, Integer::sum);     // add 5 to existing
map.replaceAll((k, v) -> v * 2);         // transform all values

// LinkedHashMap — maintains insertion order
Map<String, Integer> ordered = new LinkedHashMap<>();

// TreeMap — sorted by key
Map<String, Integer> sorted = new TreeMap<>();

// Immutable (Java 9+)
Map<String, Integer> immutable = Map.of("A", 1, "B", 2, "C", 3);
Map<String, Integer> immutable2 = Map.ofEntries(
    Map.entry("A", 1),
    Map.entry("B", 2)
);
```

### Queue & Deque
```java
// Queue — FIFO
Queue<String> queue = new LinkedList<>();
queue.offer("A");         // add (returns false if full)
queue.add("B");           // add (throws if full)
queue.peek();             // view front (null if empty)
queue.element();          // view front (throws if empty)
queue.poll();             // remove front (null if empty)
queue.remove();           // remove front (throws if empty)

// PriorityQueue — min-heap by default
PriorityQueue<Integer> pq = new PriorityQueue<>();
PriorityQueue<Integer> maxPq = new PriorityQueue<>(Comparator.reverseOrder());
pq.offer(5); pq.offer(1); pq.offer(3);
pq.peek();   // 1 (smallest)
pq.poll();   // 1

// Deque — double-ended queue
Deque<String> deque = new ArrayDeque<>();
deque.offerFirst("A");    // add to front
deque.offerLast("B");     // add to back
deque.peekFirst();        // view front
deque.peekLast();         // view back
deque.pollFirst();        // remove front
deque.pollLast();         // remove back
// Use Deque as Stack: push(), pop(), peek()
deque.push("A");
deque.pop();
```

### Collections Utility Class
```java
import java.util.Collections;

Collections.sort(list);
Collections.reverse(list);
Collections.shuffle(list);
Collections.swap(list, 0, 3);
Collections.rotate(list, 2);
Collections.fill(list, "X");
Collections.frequency(list, "A");     // count
Collections.min(list);
Collections.max(list);
Collections.binarySearch(list, "key"); // list must be sorted
Collections.disjoint(list1, list2);   // no common elements?
Collections.nCopies(5, "X");          // immutable list of 5 "X"s
Collections.singletonList("X");       // immutable single-element list
Collections.unmodifiableList(list);   // read-only wrapper
Collections.synchronizedList(list);   // thread-safe wrapper
Collections.emptyList();              // empty immutable list
```

---

## 23. GENERICS

### Generic Class
```java
public class Box<T> {
    private T content;

    public Box(T content) { this.content = content; }
    public T get() { return content; }
    public void set(T content) { this.content = content; }
}

Box<String> strBox = new Box<>("Hello");
Box<Integer> intBox = new Box<>(42);
String s = strBox.get();    // no cast needed
```

### Generic Method
```java
public <T> void printArray(T[] array) {
    for (T element : array) {
        System.out.println(element);
    }
}

public <T extends Comparable<T>> T max(T a, T b) {
    return a.compareTo(b) > 0 ? a : b;
}

// Usage — type inferred
printArray(new String[]{"A", "B", "C"});
max(3, 5);      // works with Integer (autoboxing)
max("abc", "xyz");
```

### Generic Interface
```java
public interface Repository<T, ID> {
    T findById(ID id);
    List<T> findAll();
    void save(T entity);
    void delete(ID id);
}

public class UserRepo implements Repository<User, Long> {
    @Override
    public User findById(Long id) { ... }
    // ...
}
```

### Bounded Type Parameters
```java
// Upper bound — T must extend Number
public <T extends Number> double sum(List<T> list) {
    double total = 0;
    for (T num : list) total += num.doubleValue();
    return total;
}

// Multiple bounds (class first, then interfaces)
public <T extends Comparable<T> & Serializable> void process(T item) { }
```

### Wildcards
```java
// Unbounded wildcard
void printList(List<?> list) {
    for (Object item : list) System.out.println(item);
}

// Upper bounded — ? extends Type (read only — "producer")
void sum(List<? extends Number> list) {
    double total = 0;
    for (Number n : list) total += n.doubleValue();
}
// Works with List<Integer>, List<Double>, etc.

// Lower bounded — ? super Type (write only — "consumer")
void addInts(List<? super Integer> list) {
    list.add(1);
    list.add(2);
}
// Works with List<Integer>, List<Number>, List<Object>

// PECS: Producer Extends, Consumer Super
```

### Type Erasure
- Generics are a **compile-time** feature only.
- At runtime, `List<String>` becomes just `List` (raw type).
- Cannot do: `new T()`, `new T[]`, `instanceof T`, `T.class`

---

## 24. LAMBDA EXPRESSIONS (Java 8+)

### Syntax
```java
// Full syntax
(parameters) -> { statements; }

// Single expression (no braces, implicit return)
(a, b) -> a + b

// Single parameter (no parentheses)
x -> x * 2

// No parameters
() -> System.out.println("Hello")

// With type declarations
(int a, int b) -> a + b
```

### Examples
```java
// Runnable
Runnable task = () -> System.out.println("Running");

// Comparator
Comparator<String> byLength = (a, b) -> a.length() - b.length();
list.sort((a, b) -> a.compareTo(b));

// Custom functional interface
@FunctionalInterface
interface MathOp {
    int operate(int a, int b);
}
MathOp add = (a, b) -> a + b;
MathOp mul = (a, b) -> a * b;
add.operate(3, 5);    // 8
```

### Built-in Functional Interfaces (java.util.function)
| Interface | Method | Signature |
|-----------|--------|-----------|
| `Function<T,R>` | `apply` | T → R |
| `BiFunction<T,U,R>` | `apply` | (T, U) → R |
| `Predicate<T>` | `test` | T → boolean |
| `BiPredicate<T,U>` | `test` | (T, U) → boolean |
| `Consumer<T>` | `accept` | T → void |
| `BiConsumer<T,U>` | `accept` | (T, U) → void |
| `Supplier<T>` | `get` | () → T |
| `UnaryOperator<T>` | `apply` | T → T |
| `BinaryOperator<T>` | `apply` | (T, T) → T |

```java
Function<String, Integer> strLen = String::length;
Predicate<String> isEmpty = String::isEmpty;
Consumer<String> printer = System.out::println;
Supplier<List<String>> listFactory = ArrayList::new;
UnaryOperator<String> upper = String::toUpperCase;
BinaryOperator<Integer> max = Integer::max;

// Chaining
Predicate<String> notEmpty = isEmpty.negate();
Predicate<String> shortAndNotEmpty = notEmpty.and(s -> s.length() < 10);
Function<String, String> trimAndUpper = String::trim.andThen(String::toUpperCase);
```

### Method References
```java
// Static method reference
Function<String, Integer> parse = Integer::parseInt;

// Instance method of specific object
Consumer<String> printer = System.out::println;

// Instance method of arbitrary object
Function<String, String> upper = String::toUpperCase;

// Constructor reference
Supplier<List<String>> listMaker = ArrayList::new;
Function<String, Person> personMaker = Person::new;
```

---

## 25. STREAMS API (Java 8+)

### Creating Streams
```java
// From collection
List<String> list = List.of("a", "b", "c");
Stream<String> stream = list.stream();
Stream<String> parallel = list.parallelStream();

// From values
Stream<String> stream = Stream.of("a", "b", "c");

// From array
int[] arr = {1, 2, 3};
IntStream intStream = Arrays.stream(arr);

// Generate
Stream<Double> randoms = Stream.generate(Math::random).limit(5);
Stream<Integer> nums = Stream.iterate(0, n -> n + 2).limit(10);  // 0,2,4,...
Stream<Integer> bounded = Stream.iterate(0, n -> n < 100, n -> n + 10);  // Java 9+

// Range
IntStream.range(0, 10)       // 0 to 9
IntStream.rangeClosed(1, 10)  // 1 to 10

// From string
"hello".chars()              // IntStream

// Empty
Stream.empty()

// Concat
Stream.concat(stream1, stream2)
```

### Intermediate Operations (Lazy — return Stream)
```java
List<String> names = List.of("Alice", "Bob", "Charlie", "Diana", "Eve");

names.stream()
    .filter(n -> n.length() > 3)        // keep matching elements
    .map(String::toUpperCase)           // transform each element
    .sorted()                           // natural order
    .sorted(Comparator.reverseOrder())  // custom order
    .distinct()                         // remove duplicates
    .limit(3)                           // take first 3
    .skip(1)                            // skip first 1
    .peek(System.out::println)         // debug (view without consuming)
    .flatMap(s -> s.chars().boxed())    // flatten nested streams
    .mapToInt(String::length)           // to IntStream
    .mapToDouble(...)                   // to DoubleStream
    .takeWhile(n -> n.length() < 5)    // Java 9+ — take while true
    .dropWhile(n -> n.length() < 5)    // Java 9+ — drop while true
    ;
```

### Terminal Operations (Trigger execution)
```java
// forEach — iterate
names.stream().forEach(System.out::println);

// collect — gather into collection
List<String> result = names.stream()
    .filter(n -> n.length() > 3)
    .collect(Collectors.toList());

// toList() shorthand (Java 16+, returns unmodifiable)
List<String> result = names.stream().filter(...).toList();

// collect to different types
Set<String> set = stream.collect(Collectors.toSet());
Map<String, Integer> map = stream.collect(Collectors.toMap(n -> n, String::length));
String joined = stream.collect(Collectors.joining(", "));
String joined = stream.collect(Collectors.joining(", ", "[", "]"));

// Reduce
int sum = IntStream.rangeClosed(1, 10).reduce(0, Integer::sum);
Optional<String> longest = names.stream().reduce((a, b) -> a.length() > b.length() ? a : b);

// Count, min, max
long count = names.stream().count();
Optional<String> min = names.stream().min(Comparator.naturalOrder());
Optional<String> max = names.stream().max(Comparator.comparing(String::length));

// Match
boolean any = names.stream().anyMatch(n -> n.startsWith("A"));     // true
boolean all = names.stream().allMatch(n -> n.length() > 2);        // true
boolean none = names.stream().noneMatch(n -> n.isEmpty());         // true

// Find
Optional<String> first = names.stream().filter(n -> n.startsWith("C")).findFirst();
Optional<String> any2 = names.stream().findAny();

// toArray
String[] arr = names.stream().toArray(String[]::new);

// Statistics (IntStream, DoubleStream, LongStream)
IntSummaryStatistics stats = names.stream().mapToInt(String::length).summaryStatistics();
stats.getMax();
stats.getMin();
stats.getAverage();
stats.getSum();
stats.getCount();
```

### Collectors
```java
import java.util.stream.Collectors;

// groupingBy
Map<Integer, List<String>> byLength = names.stream()
    .collect(Collectors.groupingBy(String::length));

// groupingBy with downstream
Map<Integer, Long> countByLength = names.stream()
    .collect(Collectors.groupingBy(String::length, Collectors.counting()));

// partitioningBy
Map<Boolean, List<String>> partition = names.stream()
    .collect(Collectors.partitioningBy(n -> n.length() > 3));

// summarizing
DoubleSummaryStatistics stats = items.stream()
    .collect(Collectors.summarizingDouble(Item::getPrice));

// mapping
Map<String, List<String>> deptToNames = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDept,
        Collectors.mapping(Employee::getName, Collectors.toList())
    ));

// toUnmodifiableList / toUnmodifiableSet / toUnmodifiableMap (Java 10+)
List<String> unmod = stream.collect(Collectors.toUnmodifiableList());

// teeing — combine two collectors (Java 12+)
var result = stream.collect(Collectors.teeing(
    Collectors.counting(),
    Collectors.summingInt(Integer::intValue),
    (count, sum) -> "Count: " + count + ", Sum: " + sum
));
```

### Parallel Streams
```java
list.parallelStream()
    .filter(...)
    .map(...)
    .collect(Collectors.toList());

// Or convert existing stream
list.stream().parallel()...

// Use for CPU-intensive tasks with large data
// AVOID for: IO operations, shared mutable state, small datasets, order-dependent ops
```

---

## 26. OPTIONAL (Java 8+)

```java
import java.util.Optional;

// Create
Optional<String> opt = Optional.of("Hello");         // non-null value
Optional<String> opt = Optional.ofNullable(value);   // possibly null
Optional<String> opt = Optional.empty();             // empty

// Check and get
opt.isPresent()              // true if value exists
opt.isEmpty()                // true if empty (Java 11+)
opt.get()                    // get value (throws NoSuchElementException if empty)

// Safe access
opt.orElse("default")                    // value or default
opt.orElseGet(() -> computeDefault())    // lazy default
opt.orElseThrow()                        // throw NoSuchElementException
opt.orElseThrow(() -> new RuntimeException("Not found"))

// Transform
opt.map(String::toUpperCase)             // Optional<String>
opt.flatMap(s -> findById(s))            // when function returns Optional
opt.filter(s -> s.length() > 3)          // Optional, empty if false

// Consume
opt.ifPresent(System.out::println)
opt.ifPresentOrElse(                     // Java 9+
    System.out::println,
    () -> System.out.println("empty")
);

// Stream (Java 9+)
opt.stream()                              // Stream with 0 or 1 element
opt.or(() -> Optional.of("fallback"))     // chain Optionals (Java 9+)

// Pattern
public Optional<User> findByName(String name) {
    User user = database.query(name);
    return Optional.ofNullable(user);
}

findByName("Ritesh")
    .map(User::getEmail)
    .filter(email -> email.contains("@"))
    .ifPresent(email -> sendNotification(email));
```

---

## 27. FILE I/O

### Classic I/O (java.io)
```java
import java.io.*;

// Read text file
try (BufferedReader reader = new BufferedReader(new FileReader("file.txt"))) {
    String line;
    while ((line = reader.readLine()) != null) {
        System.out.println(line);
    }
}

// Write text file
try (BufferedWriter writer = new BufferedWriter(new FileWriter("file.txt"))) {
    writer.write("Hello");
    writer.newLine();
    writer.write("World");
}

// Append
try (FileWriter writer = new FileWriter("file.txt", true)) {
    writer.write("Appended\n");
}

// Read all at once (Java 7+ — scanner)
try (Scanner scanner = new Scanner(new File("file.txt"))) {
    while (scanner.hasNextLine()) {
        System.out.println(scanner.nextLine());
    }
}

// Binary files
try (FileInputStream fis = new FileInputStream("image.png");
     FileOutputStream fos = new FileOutputStream("copy.png")) {
    byte[] buffer = new byte[8192];
    int bytesRead;
    while ((bytesRead = fis.read(buffer)) != -1) {
        fos.write(buffer, 0, bytesRead);
    }
}

// PrintWriter — convenient writing
try (PrintWriter pw = new PrintWriter(new FileWriter("file.txt"))) {
    pw.println("Line 1");
    pw.printf("Name: %s, Age: %d%n", "Ritesh", 25);
}
```

### NIO.2 (java.nio.file — Modern, Recommended)
```java
import java.nio.file.*;
import java.nio.charset.StandardCharsets;

Path path = Path.of("file.txt");
// or: Paths.get("file.txt")

// Read
String content = Files.readString(path);                     // Java 11+
List<String> lines = Files.readAllLines(path);
byte[] bytes = Files.readAllBytes(path);
Stream<String> lineStream = Files.lines(path);               // lazy

// Write
Files.writeString(path, "Hello\nWorld");                     // Java 11+
Files.write(path, List.of("Line 1", "Line 2"));
Files.write(path, "Appended".getBytes(), StandardOpenOption.APPEND);

// Copy, Move, Delete
Files.copy(source, target, StandardCopyOption.REPLACE_EXISTING);
Files.move(source, target, StandardCopyOption.ATOMIC_MOVE);
Files.delete(path);                   // throws if not exists
Files.deleteIfExists(path);

// Create
Files.createFile(path);
Files.createDirectory(Path.of("newdir"));
Files.createDirectories(Path.of("a/b/c"));    // including parents
Path tmp = Files.createTempFile("prefix", ".tmp");
Path tmpDir = Files.createTempDirectory("prefix");

// Check
Files.exists(path)
Files.notExists(path)
Files.isReadable(path)
Files.isWritable(path)
Files.isExecutable(path)
Files.isDirectory(path)
Files.isRegularFile(path)
Files.isSymbolicLink(path)

// Info
Files.size(path)                     // bytes
Files.getLastModifiedTime(path)
Files.getOwner(path)
Files.probeContentType(path)         // MIME type

// List directory
try (Stream<Path> entries = Files.list(Path.of("."))) {
    entries.forEach(System.out::println);
}

// Walk directory tree
try (Stream<Path> walk = Files.walk(Path.of("."))) {
    walk.filter(Files.isRegularFile(p))
        .forEach(System.out::println);
}

// Find files
try (Stream<Path> found = Files.find(Path.of("."), 10,
        (p, attr) -> p.toString().endsWith(".java"))) {
    found.forEach(System.out::println);
}
```

### Path Operations
```java
Path p = Path.of("/home/user/docs/file.txt");

p.getFileName()       // file.txt
p.getParent()         // /home/user/docs
p.getRoot()           // /
p.getNameCount()      // 4
p.getName(0)          // home
p.subpath(1, 3)       // user/docs
p.toAbsolutePath()
p.normalize()          // resolve . and ..
p.resolve("sub/file")  // append
p.relativize(other)    // relative path between
p.toFile()             // convert to File
p.toString()
```

---

## 28. CONCURRENCY & MULTITHREADING

### Creating Threads
```java
// Method 1: Extend Thread
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Thread: " + getName());
    }
}
new MyThread().start();

// Method 2: Implement Runnable
class MyTask implements Runnable {
    @Override
    public void run() {
        System.out.println("Running");
    }
}
new Thread(new MyTask()).start();

// Method 3: Lambda
Thread t = new Thread(() -> System.out.println("Lambda thread"));
t.start();

// Method 4: Virtual Threads (Java 21+)
Thread.startVirtualThread(() -> System.out.println("Virtual"));
Thread vThread = Thread.ofVirtual().start(() -> { ... });
```

### Thread Methods
```java
t.start()            // start execution
t.join()             // wait for thread to finish
t.join(1000)         // wait with timeout (ms)
t.isAlive()          // still running?
t.getName()          // thread name
t.setName("MyThread")
t.setPriority(Thread.MAX_PRIORITY)  // 1-10
t.setDaemon(true)    // daemon thread (dies with main)
t.interrupt()        // request interruption

Thread.currentThread()   // current thread reference
Thread.sleep(1000)       // sleep (ms)
Thread.yield()           // hint to scheduler
```

### Synchronization
```java
// synchronized method
public synchronized void increment() {
    count++;
}

// synchronized block (more granular)
public void addItem(String item) {
    synchronized (this) {
        list.add(item);
    }
}

// synchronized on specific lock
private final Object lock = new Object();
public void update() {
    synchronized (lock) {
        // critical section
    }
}
```

### Locks (java.util.concurrent.locks)
```java
import java.util.concurrent.locks.*;

ReentrantLock lock = new ReentrantLock();

lock.lock();
try {
    // critical section
} finally {
    lock.unlock();    // always in finally!
}

// tryLock
if (lock.tryLock(5, TimeUnit.SECONDS)) {
    try { ... } finally { lock.unlock(); }
}

// ReadWriteLock — multiple readers, single writer
ReadWriteLock rwLock = new ReentrantReadWriteLock();
rwLock.readLock().lock();      // shared read
rwLock.writeLock().lock();     // exclusive write
```

### ExecutorService (Thread Pool)
```java
import java.util.concurrent.*;

// Fixed thread pool
ExecutorService executor = Executors.newFixedThreadPool(4);

// Submit Runnable
executor.submit(() -> System.out.println("Task 1"));

// Submit Callable (returns Future)
Future<Integer> future = executor.submit(() -> {
    Thread.sleep(1000);
    return 42;
});
int result = future.get();           // blocks until done
int result = future.get(5, TimeUnit.SECONDS);  // with timeout
future.isDone();
future.cancel(true);                 // cancel

// Invoke all
List<Callable<String>> tasks = List.of(
    () -> "Result 1",
    () -> "Result 2",
    () -> "Result 3"
);
List<Future<String>> futures = executor.invokeAll(tasks);

// Shutdown
executor.shutdown();                 // no new tasks, finish existing
executor.shutdownNow();             // attempt to stop all
executor.awaitTermination(60, TimeUnit.SECONDS);

// Other pool types
Executors.newSingleThreadExecutor()   // single thread
Executors.newCachedThreadPool()       // dynamic pool
Executors.newScheduledThreadPool(4)   // scheduled tasks
Executors.newVirtualThreadPerTaskExecutor()  // Java 21+
```

### CompletableFuture (Java 8+)
```java
import java.util.concurrent.CompletableFuture;

// Async execution
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    return fetchData();
});

// Chain transformations
CompletableFuture<Integer> result = CompletableFuture
    .supplyAsync(() -> "Hello")
    .thenApply(String::length)         // transform result
    .thenApply(len -> len * 2);

// Consume
future.thenAccept(System.out::println);   // consume result
future.thenRun(() -> System.out.println("Done"));  // no input

// Combine
CompletableFuture<String> combined = future1
    .thenCombine(future2, (r1, r2) -> r1 + " " + r2);

// All of / Any of
CompletableFuture.allOf(f1, f2, f3).join();
CompletableFuture.anyOf(f1, f2, f3).thenAccept(...);

// Exception handling
future.exceptionally(ex -> "Fallback");
future.handle((result, ex) -> ex != null ? "Error" : result);
future.whenComplete((result, ex) -> { ... });
```

### Concurrent Collections
```java
ConcurrentHashMap<String, Integer>    // thread-safe HashMap
CopyOnWriteArrayList<String>          // thread-safe ArrayList (read-heavy)
CopyOnWriteArraySet<String>           // thread-safe Set
ConcurrentLinkedQueue<String>         // lock-free queue
BlockingQueue<String>                  // producer-consumer
    - ArrayBlockingQueue
    - LinkedBlockingQueue
    - PriorityBlockingQueue
ConcurrentSkipListMap<K,V>            // thread-safe sorted map
```

### Atomic Variables
```java
import java.util.concurrent.atomic.*;

AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet();        // ++count (atomic)
count.getAndIncrement();        // count++ (atomic)
count.addAndGet(5);
count.compareAndSet(5, 10);     // CAS operation
count.get();

AtomicLong, AtomicBoolean, AtomicReference<T>
```

---

## 29. DATE & TIME (java.time — Java 8+)

```java
import java.time.*;
import java.time.format.DateTimeFormatter;
import java.time.temporal.ChronoUnit;

// Current
LocalDate today = LocalDate.now();              // 2026-04-14
LocalTime now = LocalTime.now();                // 10:30:45.123
LocalDateTime dateTime = LocalDateTime.now();   // 2026-04-14T10:30:45
ZonedDateTime zoned = ZonedDateTime.now();      // with timezone
Instant instant = Instant.now();                // UTC timestamp

// Create
LocalDate date = LocalDate.of(2026, 4, 14);
LocalDate date = LocalDate.of(2026, Month.APRIL, 14);
LocalTime time = LocalTime.of(10, 30, 0);
LocalDateTime dt = LocalDateTime.of(2026, 4, 14, 10, 30);

// Access
date.getYear()         // 2026
date.getMonth()        // APRIL
date.getMonthValue()   // 4
date.getDayOfMonth()   // 14
date.getDayOfWeek()    // MONDAY
date.getDayOfYear()    // 104
date.lengthOfMonth()   // 30
date.isLeapYear()      // false

// Arithmetic
date.plusDays(10)
date.plusWeeks(2)
date.plusMonths(3)
date.plusYears(1)
date.minusDays(5)
date.plus(Period.ofMonths(3))
time.plusHours(2)
time.plusMinutes(30)
dateTime.plus(Duration.ofHours(5))

// Between
Period period = Period.between(date1, date2);
period.getYears(); period.getMonths(); period.getDays();

Duration duration = Duration.between(time1, time2);
duration.toHours(); duration.toMinutes(); duration.toSeconds();

long daysBetween = ChronoUnit.DAYS.between(date1, date2);

// Compare
date1.isBefore(date2)
date1.isAfter(date2)
date1.isEqual(date2)

// Format
DateTimeFormatter fmt = DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm:ss");
String formatted = dateTime.format(fmt);       // "14/04/2026 10:30:00"
dateTime.format(DateTimeFormatter.ISO_LOCAL_DATE_TIME);

// Parse
LocalDate parsed = LocalDate.parse("2026-04-14");
LocalDate parsed = LocalDate.parse("14/04/2026", DateTimeFormatter.ofPattern("dd/MM/yyyy"));

// Timezone
ZoneId zone = ZoneId.of("Asia/Kolkata");
ZonedDateTime zdt = ZonedDateTime.now(zone);
ZonedDateTime zdt = dateTime.atZone(zone);
Set<String> zones = ZoneId.getAvailableZoneIds();

// Instant (machine timestamp)
Instant now = Instant.now();
long epoch = now.toEpochMilli();
Instant from = Instant.ofEpochMilli(epoch);
```

---

## 30. REGULAR EXPRESSIONS

```java
import java.util.regex.*;

String text = "My phone is 123-456-7890";

// Compile pattern
Pattern pattern = Pattern.compile("\\d{3}-\\d{3}-\\d{4}");
Matcher matcher = pattern.matcher(text);

// Find
if (matcher.find()) {
    System.out.println(matcher.group());    // "123-456-7890"
    System.out.println(matcher.start());    // start index
    System.out.println(matcher.end());      // end index
}

// Find all
while (matcher.find()) {
    System.out.println(matcher.group());
}

// Matches (entire string must match)
boolean matches = Pattern.matches("\\d+", "12345");
boolean matches = "12345".matches("\\d+");

// Groups
Pattern p = Pattern.compile("(\\d{3})-(\\d{3})-(\\d{4})");
Matcher m = p.matcher("123-456-7890");
if (m.find()) {
    m.group(0)   // "123-456-7890" (whole match)
    m.group(1)   // "123"
    m.group(2)   // "456"
    m.group(3)   // "7890"
}

// Named groups
Pattern p = Pattern.compile("(?<area>\\d{3})-(?<prefix>\\d{3})-(?<line>\\d{4})");
if (m.find()) {
    m.group("area")    // "123"
}

// Replace
String result = text.replaceAll("\\d", "*");
String result = matcher.replaceAll("***");
String result = matcher.replaceFirst("***");

// Split
String[] parts = "a,b,,c".split(",");       // ["a","b","","c"]
String[] parts = "a,b,,c".split(",", -1);   // keep trailing empty
Pattern.compile(",").split("a,b,c");

// Flags
Pattern.compile("hello", Pattern.CASE_INSENSITIVE);
Pattern.compile("hello", Pattern.CASE_INSENSITIVE | Pattern.MULTILINE);

// Quick one-liners
"email@test.com".matches("[\\w.]+@[\\w.]+");
"hello123".replaceAll("[^a-zA-Z]", "");  // "hello"

// Predicate (Java 8+)
Pattern.compile("\\d+").asPredicate().test("123");  // true

// Stream (Java 9+)
Pattern.compile(",").splitAsStream("a,b,c").forEach(System.out::println);
```

---

## 31. ANNOTATIONS

### Built-in Annotations
```java
@Override           // method overrides superclass method
@Deprecated         // marks as deprecated
@Deprecated(since = "9", forRemoval = true)
@SuppressWarnings("unchecked")
@SuppressWarnings({"unused", "deprecation"})
@FunctionalInterface  // interface has exactly one abstract method
@SafeVarargs        // suppress heap pollution warnings for varargs
```

### Custom Annotations
```java
import java.lang.annotation.*;

@Retention(RetentionPolicy.RUNTIME)     // available at runtime
@Target(ElementType.METHOD)              // can be applied to methods
@Documented                              // included in Javadoc
@Inherited                               // subclasses inherit it
public @interface MyAnnotation {
    String value() default "";
    int priority() default 0;
    String[] tags() default {};
}

@MyAnnotation(value = "test", priority = 1, tags = {"a", "b"})
public void myMethod() { }

// Reading annotations at runtime (reflection)
Method method = MyClass.class.getMethod("myMethod");
MyAnnotation ann = method.getAnnotation(MyAnnotation.class);
ann.value();     // "test"
ann.priority();  // 1
```

### Retention Policies
| Policy | Description |
|--------|-------------|
| `SOURCE` | Discarded by compiler |
| `CLASS` | In .class file, not at runtime |
| `RUNTIME` | Available at runtime via reflection |

### Target Types
| Target | Where |
|--------|-------|
| `TYPE` | Class, interface, enum, record |
| `FIELD` | Field |
| `METHOD` | Method |
| `PARAMETER` | Method parameter |
| `CONSTRUCTOR` | Constructor |
| `LOCAL_VARIABLE` | Local variable |
| `ANNOTATION_TYPE` | Another annotation |
| `PACKAGE` | Package |
| `TYPE_PARAMETER` | Generic type parameter |
| `TYPE_USE` | Any type use |

---

## 32. REFLECTION API

### What is Reflection?
- **Reflection** allows inspecting and manipulating classes, methods, fields, constructors **at runtime**.
- Lives in the `java.lang.reflect` package.
- Used by frameworks (Spring, Hibernate, JUnit), serialization libraries, IDE tools, debuggers.
- Comes with **performance overhead** — use only when necessary.

### Getting a Class Object
```java
// 3 ways to get a Class object
Class<?> c1 = String.class;                          // from type literal
Class<?> c2 = "hello".getClass();                    // from instance
Class<?> c3 = Class.forName("java.lang.String");     // from fully-qualified name (can throw ClassNotFoundException)
```

### Inspecting Class Metadata
```java
Class<?> clazz = MyClass.class;

clazz.getName();              // "com.example.MyClass"
clazz.getSimpleName();        // "MyClass"
clazz.getCanonicalName();     // "com.example.MyClass"
clazz.getPackageName();       // "com.example"
clazz.getSuperclass();        // parent class
clazz.getInterfaces();        // implemented interfaces
clazz.getModifiers();         // returns int — decode with Modifier.isPublic(), etc.
clazz.isInterface();          // true if interface
clazz.isEnum();               // true if enum
clazz.isArray();              // true if array
clazz.isAnnotation();         // true if annotation type
Modifier.isAbstract(clazz.getModifiers());   // check abstract
```

### Working with Fields
```java
import java.lang.reflect.Field;

Class<?> clazz = MyClass.class;

// All public fields (including inherited)
Field[] publicFields = clazz.getFields();

// All declared fields (public + private + protected, this class only)
Field[] allFields = clazz.getDeclaredFields();

// Single field by name
Field field = clazz.getDeclaredField("name");

// Access private fields
field.setAccessible(true);     // bypass access check

// Get & set values
Object obj = new MyClass();
Object value = field.get(obj);        // get field value
field.set(obj, "newValue");           // set field value

// Field info
field.getName();                      // "name"
field.getType();                      // String.class
field.getModifiers();                 // int — use Modifier.isPrivate(), etc.
field.isAnnotationPresent(MyAnno.class);  // check annotation
```

### Working with Methods
```java
import java.lang.reflect.Method;

Class<?> clazz = MyClass.class;

// All public methods (including inherited from Object)
Method[] publicMethods = clazz.getMethods();

// All declared methods (this class only)
Method[] allMethods = clazz.getDeclaredMethods();

// Single method by name + parameter types
Method method = clazz.getDeclaredMethod("greet", String.class);

// Invoke method
method.setAccessible(true);                        // if private
Object result = method.invoke(obj, "Ritesh");      // invoke on instance

// Static method invocation
Method staticMethod = clazz.getDeclaredMethod("staticHello");
staticMethod.invoke(null);                         // null for static methods

// Method info
method.getName();                    // "greet"
method.getReturnType();              // String.class
method.getParameterTypes();          // Class<?>[] — parameter types
method.getParameterCount();          // number of params
method.getExceptionTypes();          // declared checked exceptions
method.isVarArgs();                  // true if varargs
```

### Working with Constructors
```java
import java.lang.reflect.Constructor;

Class<?> clazz = MyClass.class;

// All public constructors
Constructor<?>[] publicCtors = clazz.getConstructors();

// All declared constructors
Constructor<?>[] allCtors = clazz.getDeclaredConstructors();

// Specific constructor by param types
Constructor<?> ctor = clazz.getDeclaredConstructor(String.class, int.class);

// Create instance via constructor
ctor.setAccessible(true);
Object instance = ctor.newInstance("Ritesh", 25);

// No-arg constructor shortcut (class must have public no-arg ctor)
Object instance2 = clazz.getDeclaredConstructor().newInstance();
```

### Creating Instances Dynamically
```java
// Using Class.forName + newInstance (useful for plugin loading)
Class<?> clazz = Class.forName("com.example.MyPlugin");
Object plugin = clazz.getDeclaredConstructor().newInstance();
```

### Working with Arrays via Reflection
```java
import java.lang.reflect.Array;

// Create array dynamically
Object arr = Array.newInstance(int.class, 5);   // int[5]

// Set & get elements
Array.set(arr, 0, 42);
int val = (int) Array.get(arr, 0);             // 42

// Get array length
int len = Array.getLength(arr);                // 5

// Check if type is array
int[].class.isArray();                         // true
int[].class.getComponentType();                // int.class
```

### Reading Annotations at Runtime
```java
import java.lang.annotation.*;

// Annotation must have @Retention(RetentionPolicy.RUNTIME)
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface MyAnnotation {
    String value();
}

class Demo {
    @MyAnnotation("hello")
    public void myMethod() {}
}

// Reading at runtime
Method m = Demo.class.getMethod("myMethod");
if (m.isAnnotationPresent(MyAnnotation.class)) {
    MyAnnotation anno = m.getAnnotation(MyAnnotation.class);
    System.out.println(anno.value());    // "hello"
}

// Get all annotations
Annotation[] annotations = m.getAnnotations();
```

### Proxy — Dynamic Interface Implementation
```java
import java.lang.reflect.Proxy;
import java.lang.reflect.InvocationHandler;

interface Greeter {
    String greet(String name);
}

// Create a dynamic proxy
Greeter proxy = (Greeter) Proxy.newProxyInstance(
    Greeter.class.getClassLoader(),
    new Class<?>[] { Greeter.class },
    (Object proxyObj, Method method, Object[] args) -> {
        System.out.println("Method called: " + method.getName());
        return "Hello, " + args[0];
    }
);

System.out.println(proxy.greet("Ritesh"));  // "Hello, Ritesh"
```

### Checking & Modifying Access
```java
import java.lang.reflect.Modifier;

Field field = clazz.getDeclaredField("secret");

// Check modifiers
int mods = field.getModifiers();
Modifier.isPrivate(mods);     // true
Modifier.isStatic(mods);      // false
Modifier.isFinal(mods);       // false
Modifier.isTransient(mods);   // false

// Bypass access control (use carefully)
field.setAccessible(true);    // allows reading/writing private fields
```

### Inspecting Generics via Reflection
```java
import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;

class MyClass {
    private List<String> names;
}

Field f = MyClass.class.getDeclaredField("names");
Type genericType = f.getGenericType();

if (genericType instanceof ParameterizedType pt) {
    System.out.println(pt.getRawType());              // interface java.util.List
    System.out.println(pt.getActualTypeArguments()[0]); // class java.lang.String
}
```

### Reflection Quick Reference
| Task | Method |
|------|--------|
| Get class object | `Class.forName("...")`, `.class`, `.getClass()` |
| List fields | `getFields()`, `getDeclaredFields()` |
| List methods | `getMethods()`, `getDeclaredMethods()` |
| List constructors | `getConstructors()`, `getDeclaredConstructors()` |
| Create instance | `ctor.newInstance(args...)` |
| Invoke method | `method.invoke(obj, args...)` |
| Get/set field value | `field.get(obj)`, `field.set(obj, val)` |
| Access private members | `setAccessible(true)` |
| Read annotations | `getAnnotation(A.class)`, `isAnnotationPresent(A.class)` |
| Dynamic proxy | `Proxy.newProxyInstance(...)` |
| Inspect generics | `getGenericType()` → `ParameterizedType` |

### Reflection — When to Use & When to Avoid
| Use When | Avoid When |
|----------|------------|
| Building frameworks / DI containers | Regular application logic |
| Serialization / deserialization libraries | Performance-critical code |
| Testing private methods (unit tests) | Security-sensitive contexts |
| Plugin / module loading at runtime | When compile-time types suffice |
| ORM mapping (Hibernate, JPA) | Simple object creation |

> **⚠ Caution:** Reflection breaks encapsulation, bypasses compile-time checks, and is slower than direct access. Prefer direct code unless you genuinely need runtime introspection.

---

## 33. MODULES (Java 9+)

```java
// module-info.java (in module root)
module com.myapp {
    requires java.sql;                  // dependency
    requires transitive java.logging;   // transitive dependency
    requires static lombok;             // compile-time only

    exports com.myapp.api;              // make package public
    exports com.myapp.internal to com.myapp.test;  // qualified

    opens com.myapp.model;              // allow reflection
    opens com.myapp.model to com.google.gson;

    provides com.myapp.spi.MyService
        with com.myapp.impl.MyServiceImpl;   // service provider

    uses com.myapp.spi.MyService;       // service consumer
}
```

---

## 34. INPUT / OUTPUT STREAMS SUMMARY

| Class | Purpose |
|-------|---------|
| `InputStream` / `OutputStream` | Abstract byte streams |
| `FileInputStream` / `FileOutputStream` | File byte I/O |
| `BufferedInputStream` / `BufferedOutputStream` | Buffered byte I/O |
| `DataInputStream` / `DataOutputStream` | Primitive type I/O |
| `ObjectInputStream` / `ObjectOutputStream` | Object serialization |
| `ByteArrayInputStream` / `ByteArrayOutputStream` | Byte array I/O |
| `Reader` / `Writer` | Abstract character streams |
| `FileReader` / `FileWriter` | File character I/O |
| `BufferedReader` / `BufferedWriter` | Buffered character I/O |
| `InputStreamReader` / `OutputStreamWriter` | Byte → Char bridge |
| `PrintWriter` / `PrintStream` | Convenient formatted output |
| `StringReader` / `StringWriter` | String-based I/O |
| `Scanner` | Tokenized input parsing |

### Serialization
```java
// Class must implement Serializable
public class User implements Serializable {
    private static final long serialVersionUID = 1L;
    private String name;
    private transient String password;   // not serialized
}

// Write
try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("user.ser"))) {
    oos.writeObject(user);
}

// Read
try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("user.ser"))) {
    User user = (User) ois.readObject();
}
```

---

## 35. KEY DESIGN PATTERNS

### Singleton
```java
public class Singleton {
    private static final Singleton INSTANCE = new Singleton();
    private Singleton() { }
    public static Singleton getInstance() { return INSTANCE; }
}

// Thread-safe lazy
public class Singleton {
    private static volatile Singleton instance;
    private Singleton() { }
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) instance = new Singleton();
            }
        }
        return instance;
    }
}

// Best approach — enum
public enum Singleton {
    INSTANCE;
    public void doWork() { }
}
```

### Builder
```java
public class User {
    private final String name;
    private final int age;
    private final String email;

    private User(Builder builder) {
        this.name = builder.name;
        this.age = builder.age;
        this.email = builder.email;
    }

    public static class Builder {
        private final String name;
        private int age;
        private String email;

        public Builder(String name) { this.name = name; }
        public Builder age(int age) { this.age = age; return this; }
        public Builder email(String email) { this.email = email; return this; }
        public User build() { return new User(this); }
    }
}

User user = new User.Builder("Ritesh").age(25).email("r@test.com").build();
```

### Factory
```java
public interface Shape { void draw(); }
public class Circle implements Shape { public void draw() { ... } }
public class Rectangle implements Shape { public void draw() { ... } }

public class ShapeFactory {
    public static Shape create(String type) {
        return switch (type.toLowerCase()) {
            case "circle" -> new Circle();
            case "rectangle" -> new Rectangle();
            default -> throw new IllegalArgumentException("Unknown: " + type);
        };
    }
}
Shape s = ShapeFactory.create("circle");
```

### Observer
```java
// Java built-in: PropertyChangeListener, or implement manually
interface Observer { void update(String event, Object data); }
interface Observable { void subscribe(Observer o); void notify(String event, Object data); }
```

### Strategy
```java
interface SortStrategy { void sort(int[] array); }
class BubbleSort implements SortStrategy { public void sort(int[] arr) { ... } }
class QuickSort implements SortStrategy { public void sort(int[] arr) { ... } }

class Sorter {
    private SortStrategy strategy;
    public Sorter(SortStrategy strategy) { this.strategy = strategy; }
    public void sort(int[] arr) { strategy.sort(arr); }
}

// With lambdas
Sorter sorter = new Sorter(arr -> Arrays.sort(arr));
```

---

## 36. USEFUL CLASSES & UTILITIES

### Math
```java
Math.abs(-5)              // 5
Math.max(3, 7)            // 7
Math.min(3, 7)            // 3
Math.pow(2, 10)           // 1024.0
Math.sqrt(144)            // 12.0
Math.cbrt(27)             // 3.0
Math.ceil(3.2)            // 4.0
Math.floor(3.8)           // 3.0
Math.round(3.5)           // 4
Math.random()             // [0.0, 1.0)
Math.log(Math.E)          // 1.0
Math.log10(100)           // 2.0
Math.PI                   // 3.14159...
Math.E                    // 2.71828...
Math.sin(Math.PI/2)       // 1.0
Math.toRadians(180)       // 3.14159...
Math.toDegrees(Math.PI)   // 180.0
Math.signum(-5)           // -1.0
Math.addExact(a, b)       // throws on overflow
```

### Objects
```java
import java.util.Objects;
Objects.equals(a, b)                 // null-safe equals
Objects.hash(field1, field2)         // hash code
Objects.toString(obj, "default")     // null-safe toString
Objects.requireNonNull(obj)          // throw NPE if null
Objects.requireNonNull(obj, "msg")
Objects.requireNonNullElse(obj, defaultVal)  // Java 9+
Objects.isNull(obj)                  // = (obj == null)
Objects.nonNull(obj)                 // = (obj != null)
Objects.checkIndex(index, length)    // Java 9+
```

### System
```java
System.out.println()                 // stdout
System.err.println()                 // stderr
System.exit(0)                       // terminate JVM
System.currentTimeMillis()           // epoch millis
System.nanoTime()                    // high-res timer
System.getenv("PATH")               // environment variable
System.getProperty("user.home")     // system property
System.arraycopy(src, 0, dst, 0, len)  // fast array copy
System.gc()                          // suggest garbage collection
System.lineSeparator()               // OS line separator
```

### Properties
```java
import java.util.Properties;

Properties props = new Properties();

// Load from file
try (InputStream is = new FileInputStream("config.properties")) {
    props.load(is);
}

String value = props.getProperty("key");
String value = props.getProperty("key", "default");
props.setProperty("key", "value");

// Save
try (OutputStream os = new FileOutputStream("config.properties")) {
    props.store(os, "Comments");
}
```

---

## 37. JVM & MEMORY

### Memory Areas
| Area | Stores |
|------|--------|
| **Heap** | Objects, arrays (shared by all threads) |
| **Stack** | Local variables, method calls (per thread) |
| **Method Area** | Class metadata, static variables |
| **String Pool** | Interned strings (part of heap since Java 7) |
| **PC Register** | Current instruction (per thread) |
| **Native Stack** | Native method calls |

### Garbage Collection
```java
// Java manages memory automatically — no manual free/delete
System.gc()                // suggest GC (not guaranteed)
Runtime.getRuntime().gc()
Runtime.getRuntime().freeMemory()
Runtime.getRuntime().totalMemory()
Runtime.getRuntime().maxMemory()

// GC algorithms (set via JVM flags)
// -XX:+UseG1GC         (default since Java 9)
// -XX:+UseZGC          (low latency, Java 15+)
// -XX:+UseShenandoahGC (low pause)
// -XX:+UseParallelGC   (throughput)
```

### JVM Flags
```bash
java -Xms256m -Xmx1g MyApp        # min/max heap size
java -Xss512k MyApp               # stack size
java -XX:+PrintGCDetails MyApp    # GC logs
java -XX:+HeapDumpOnOutOfMemoryError MyApp
java -ea MyApp                     # enable assertions
java -jar app.jar                  # run JAR
java -cp lib/* com.myapp.Main    # classpath
java --enable-preview MyApp       # preview features
```

---

## 38. COMMON JAVA COMMANDS

```bash
javac MyApp.java                   # compile
java MyApp                         # run
java -jar app.jar                  # run JAR
javac -d out src/*.java            # compile to directory
jar cf app.jar -C out .            # create JAR
jar tf app.jar                     # list JAR contents
jar xf app.jar                     # extract JAR
javadoc -d docs src/*.java         # generate docs
jshell                             # interactive REPL
jps                                # list Java processes
jstack <pid>                       # thread dump
jmap -heap <pid>                   # heap info
jconsole                           # monitoring GUI
jvisualvm                          # profiling GUI
jfr                                # flight recorder
keytool                            # certificate management
```

---

## 39. QUICK REFERENCE CHEAT SHEET

### Access Modifiers
```
public      → everywhere
protected   → same package + subclasses
(default)   → same package only
private     → same class only
```

### String / StringBuilder / StringBuffer
```
String        → immutable, thread-safe, use for constants
StringBuilder → mutable, NOT thread-safe, use for building strings
StringBuffer  → mutable, thread-safe (synchronized), legacy
```

### Collections Choice Guide
```
Need ordered + duplicates?          → ArrayList
Need fast insert/delete?            → LinkedList
Need unique + no order?             → HashSet
Need unique + sorted?               → TreeSet
Need unique + insertion order?      → LinkedHashSet
Need key-value?                     → HashMap
Need key-value + sorted keys?       → TreeMap
Need key-value + insertion order?   → LinkedHashMap
Need thread-safe map?               → ConcurrentHashMap
Need FIFO queue?                    → LinkedList / ArrayDeque
Need priority queue?                → PriorityQueue
Need stack (LIFO)?                  → ArrayDeque
Need thread-safe queue?             → ConcurrentLinkedQueue
```

### Exception Handling Rules
```
1. Catch specific exceptions before general ones
2. Never catch Throwable or Error
3. Always close resources (try-with-resources)
4. Don't use exceptions for flow control
5. Include original cause when re-throwing
6. Log exceptions with context
```

### Comparison Quick Reference
```
==              → reference equality (same object)
.equals()       → content equality (value comparison)
.compareTo()    → ordering (-1, 0, 1)
Objects.equals() → null-safe equality
```

### Keyword Quick Reference
```
abstract    class    extends    final      implements
import      interface  new     package    return
static      super    this       throws     void
break       continue  do       for        while
if          else     switch    case       default
try         catch    finally   throw
boolean     byte     char      double     float
int         long     short
instanceof  null     true      false
assert      enum     native    strictfp   synchronized
transient   volatile
var         yield    record    sealed     permits
non-sealed  module   exports   requires   opens
provides    uses     with      to
```

---

*Complete Java Reference — All 39 Topics*
*Last updated: April 2026*
