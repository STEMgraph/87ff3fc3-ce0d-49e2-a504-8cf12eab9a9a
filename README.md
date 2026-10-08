<!---
{
  "id": "87ff3fc3-ce0d-49e2-a504-8cf12eab9a9a",
  "teaches": "Cpp: Static Members and an Instance Counter",
  "depends_on": [
    "dd36c1f7-e644-45a8-b11b-cf150a18d38c"
  ],
  "author": "Stephan Bökelmann",
  "first_used": "2026-10-08",
  "keywords": [
    "C++",
    "static member",
    "static method",
    "class variable",
    "instance counter",
    "ODR",
    "initialization order"
  ]
}
--->

# Cpp: Static Members and an Instance Counter

> In this exercise you add a counter to a class that knows at any moment how many objects of that class exist, using a `static` attribute that is shared by all instances and lives even when there is no instance at all. You will see where it is stored, how to access it through the class name, and what goes wrong when the definition is forgotten. No `sudo` is required.

## Introduction

Every object has its own copy of each ordinary attribute. Sometimes a value belongs to the *class* rather than to any one object: the number of open connections, a lookup table shared by all instances, a next free id. C++ expresses this with `static` members.

A static attribute is declared inside the class but defined exactly once outside it; it is stored like a global variable, with the class name as its namespace. A static method can be called without an object and therefore has no `this`. The lecture's example, counting all instances of a class, needs exactly the two special methods from the previous exercise: the constructor increments, the destructor decrements.

### 1.1) Further Readings and Other Sources

* [cppreference: static members](https://en.cppreference.com/w/cpp/language/static)
* [cppreference: Static initialization order fiasco](https://en.cppreference.com/w/cpp/language/siof)
* [C++ Core Guidelines: I.3 (avoid singletons)](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Ri-singleton)

## Tasks

### Task 1: Declaration without definition

```bash
mkdir -p ~/oop-5 && cd ~/oop-5
```

```c++
// file: counter.cpp
#include <cstdio>

class Sensor {
public:
    Sensor()  { ++count; }
    ~Sensor() { --count; }
    static int count;          // declaration only
};

int main() {
    Sensor a, b;
    { Sensor c; std::printf("inside: %d\n", Sensor::count); }
    std::printf("outside: %d\n", Sensor::count);
}
```

```bash
g++ -Wall -Wextra counter.cpp -o counter
```

Read the linker error: `undefined reference to Sensor::count`. The declaration inside the class promises that the variable exists; nobody has defined it.

### Task 2: The definition

Add, outside the class:

```c++
int Sensor::count = 0;
```

Build and run. Then `nm -C counter | grep count`: the symbol is in the data segment (`B` or `D`), like a global, with the class name as prefix.

### Task 3: Access without an object

`Sensor::count` works before any `Sensor` exists. Print it as the first line of `main`. Then try `a.count` as well: allowed, but misleading; the Core Guidelines prefer the class-name form. Make the attribute `private` and add a static method:

```c++
    static int instances() { return count; }
```

Call `Sensor::instances()`. Inside a static method, try to use a non-static member and read the error: there is no `this`.

### Task 4: Next free id

```c++
class Sensor {
public:
    Sensor() : id(next_id++) { ++count; }
    int id;
private:
    static int next_id;
    static int count;
};
int Sensor::next_id = 1;
```

Create three sensors in an array and print their ids. Delete one via a scope and create another: ids are never reused, the count still matches.

### Task 5: `inline` definition (C++17)

Replace the out-of-class definition by

```c++
    inline static int count = 0;
```

Build with `g++ -std=c++17`. Now the definition can live in a header without violating the one-definition rule. Put the class into `sensor.h`, include it from two `.cpp` files, and link them; it works. Then try the same with the old non-inline definition in the header and read the "multiple definition" error from the library exercises.

### Task 6: Static initialisation order

Create two files:

```c++
// a.cpp
#include <cstdio>
int value() { return 42; }
int a = value();
```

```c++
// b.cpp
#include <cstdio>
extern int a;
int b = a + 1;
int main() { std::printf("b = %d\n", b); }
```

Link as `g++ a.cpp b.cpp` and as `g++ b.cpp a.cpp`. Depending on the order, `b` may be 1 or 43: static objects in different translation units are initialised in unspecified order. Explain why a static counter with a constant initialiser (`= 0`) is safe but a static object with a constructor can be a trap.

### Task 7: Where the counter breaks

Add `Sensor copy = a;` and print the count. It did not increase, but the destructor of `copy` will decrement it. Run to the end and observe a negative count. Fix it with a copy constructor that also increments:

```c++
    Sensor(const Sensor &other) : id(next_id++) { ++count; }
```

## Questions

- What is the difference between the declaration of `count` inside the class and its definition outside?
- Where is a static attribute stored, and how does that differ from an ordinary attribute?
- Why has a static method no `this`, and what can it therefore not do?
- What does `inline static` change about where the definition may live?
- Task 6: why is the result order dependent, and which kinds of static initialisers are immune?
- Why did the counter go negative in Task 7, and what does that tell you about the compiler-generated copy constructor?

## Advice

Static members are globals with manners: a name that says which class they belong to and access control that says who may change them. Use them for counters, caches and ids; do not use them to smuggle global state into every class ("singletons"), the Core Guidelines are explicit about that.

Task 7 is the second time a compiler-generated copy has surprised you. The rule of five chapter will close that gap; until then, when a class keeps a count or owns a resource, define its copy behaviour yourself or delete it.
