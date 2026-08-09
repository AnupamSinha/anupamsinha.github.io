---
title: "Java Collections + Lambda: Write Less, Do More"
date: 2024-08-10
categories: [Java, Fundamentals]
tags: [java, collections, lambda, streams, functional-programming]
description: "A practical guide to using Java Collections and Lambda expressions together for cleaner, shorter, and more readable code."
image:
  path: /assets/img/posts/collection_u2np.svg
  alt: Java Collections Illustration
---

![Programming](/assets/img/posts/programming_2svr.svg){: width="500" }
_Simplify your code with Collections + Lambda_

## What's This About?

If you're still writing `for` loops for everything — filtering lists, transforming data, finding items — there's a better way. Java Collections combined with Lambda expressions let you write **shorter, cleaner, more readable code**.

This guide shows real, everyday examples. No theory dumps. Just "before and after" code you can use immediately.

---

## The Basics: What is a Lambda?

A lambda is just a short way to write a function. Think of it as: **"here's what to do with each item."**

```java
// Old way — anonymous class
Comparator<String> comp = new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
        return a.compareTo(b);
    }
};

// Lambda way — same thing, one line
Comparator<String> comp = (a, b) -> a.compareTo(b);
```

That's it. Same logic, less ceremony.

---

## Filtering a List

**The task**: Get all names that start with "A" from a list.

```java
List<String> names = List.of("Anupam", "Bob", "Alice", "Charlie", "Amit");

// Old way — 5 lines
List<String> result = new ArrayList<>();
for (String name : names) {
    if (name.startsWith("A")) {
        result.add(name);
    }
}

// Lambda way — 1 line
List<String> result = names.stream()
    .filter(name -> name.startsWith("A"))
    .toList();

// Result: [Anupam, Alice, Amit]
```

> Read it as: "From names, keep only those where name starts with A."
{: .prompt-tip }

---

## Transforming Items (Map)

**The task**: Convert all names to uppercase.

```java
List<String> names = List.of("anupam", "bob", "alice");

// Old way
List<String> upper = new ArrayList<>();
for (String name : names) {
    upper.add(name.toUpperCase());
}

// Lambda way
List<String> upper = names.stream()
    .map(String::toUpperCase)
    .toList();

// Result: [ANUPAM, BOB, ALICE]
```

> Read it as: "From names, transform each one to uppercase."
{: .prompt-tip }

---

## Finding One Item

**The task**: Find the first name longer than 4 characters.

```java
List<String> names = List.of("Bob", "Alice", "Anupam", "Charlie");

// Old way
String found = null;
for (String name : names) {
    if (name.length() > 4) {
        found = name;
        break;
    }
}

// Lambda way
Optional<String> found = names.stream()
    .filter(name -> name.length() > 4)
    .findFirst();

// Result: Optional[Alice]
```

> `Optional` means "it might not exist" — no more `NullPointerException` surprises.
{: .prompt-info }

---

## Sorting

**The task**: Sort a list of people by age.

```java
record Person(String name, int age) {}

List<Person> people = List.of(
    new Person("Anupam", 35),
    new Person("Bob", 28),
    new Person("Alice", 32)
);

// Lambda way
List<Person> sorted = people.stream()
    .sorted(Comparator.comparingInt(Person::age))
    .toList();

// Result: [Bob(28), Alice(32), Anupam(35)]
```

**Sort by name instead?**

```java
people.stream()
    .sorted(Comparator.comparing(Person::name))
    .toList();
```

**Reverse order?**

```java
people.stream()
    .sorted(Comparator.comparingInt(Person::age).reversed())
    .toList();
```

---

## Removing Duplicates

```java
List<String> cities = List.of("Singapore", "Tokyo", "Singapore", "Mumbai", "Tokyo");

List<String> unique = cities.stream()
    .distinct()
    .toList();

// Result: [Singapore, Tokyo, Mumbai]
```

---

## Grouping Items

**The task**: Group people by their city.

```java
record Employee(String name, String city) {}

List<Employee> employees = List.of(
    new Employee("Anupam", "Singapore"),
    new Employee("Bob", "Tokyo"),
    new Employee("Alice", "Singapore"),
    new Employee("Charlie", "Tokyo")
);

Map<String, List<Employee>> byCity = employees.stream()
    .collect(Collectors.groupingBy(Employee::city));

// Result:
// Singapore -> [Anupam, Alice]
// Tokyo -> [Bob, Charlie]
```

> Read it as: "Group all employees by their city."
{: .prompt-tip }

---

## Counting Items

**The task**: Count how many people are in each city.

```java
Map<String, Long> countByCity = employees.stream()
    .collect(Collectors.groupingBy(Employee::city, Collectors.counting()));

// Result: {Singapore=2, Tokyo=2}
```

---

## Summing Values

**The task**: Calculate total salary.

```java
record Employee(String name, double salary) {}

List<Employee> team = List.of(
    new Employee("Anupam", 8000),
    new Employee("Bob", 6000),
    new Employee("Alice", 7500)
);

double totalSalary = team.stream()
    .mapToDouble(Employee::salary)
    .sum();

// Result: 21500.0
```

**Average salary?**

```java
double avg = team.stream()
    .mapToDouble(Employee::salary)
    .average()
    .orElse(0);

// Result: 7166.67
```

---

## Joining Strings

**The task**: Create a comma-separated string of names.

```java
List<String> names = List.of("Anupam", "Bob", "Alice");

String joined = names.stream()
    .collect(Collectors.joining(", "));

// Result: "Anupam, Bob, Alice"
```

With a prefix and suffix:

```java
String joined = names.stream()
    .collect(Collectors.joining(", ", "[", "]"));

// Result: "[Anupam, Bob, Alice]"
```

---

## Checking Conditions

### Are ALL items matching?

```java
List<Integer> scores = List.of(75, 82, 91, 68, 88);

boolean allPassed = scores.stream().allMatch(score -> score >= 60);
// true — everyone scored 60+
```

### Is ANY item matching?

```java
boolean hasTopper = scores.stream().anyMatch(score -> score >= 90);
// true — 91 exists
```

### Does NONE match?

```java
boolean noneFailed = scores.stream().noneMatch(score -> score < 40);
// true — nobody below 40
```

---

## Converting Between Collection Types

### List to Set (remove duplicates)

```java
List<String> list = List.of("A", "B", "A", "C", "B");
Set<String> set = new HashSet<>(list);
// [A, B, C]
```

### List to Map

```java
record Product(int id, String name) {}

List<Product> products = List.of(
    new Product(1, "Laptop"),
    new Product(2, "Phone"),
    new Product(3, "Tablet")
);

Map<Integer, String> productMap = products.stream()
    .collect(Collectors.toMap(Product::id, Product::name));

// {1=Laptop, 2=Phone, 3=Tablet}
```

### Map to List

```java
Map<String, Integer> scores = Map.of("Alice", 90, "Bob", 85);

List<String> toppers = scores.entrySet().stream()
    .filter(entry -> entry.getValue() >= 90)
    .map(Map.Entry::getKey)
    .toList();

// [Alice]
```

---

## Chaining Operations

![Coding](/assets/img/posts/coding_6mjf.svg){: width="450" }
_Chain multiple operations for powerful data processing_

The real power comes from **chaining** — combine filter, map, sort in one flow:

```java
record Order(String customer, double amount, String status) {}

List<Order> orders = List.of(
    new Order("Alice", 250.0, "COMPLETED"),
    new Order("Bob", 100.0, "PENDING"),
    new Order("Alice", 450.0, "COMPLETED"),
    new Order("Charlie", 300.0, "COMPLETED"),
    new Order("Bob", 200.0, "COMPLETED")
);

// Get top 3 completed orders by amount, show customer names
List<String> topCustomers = orders.stream()
    .filter(o -> o.status().equals("COMPLETED"))
    .sorted(Comparator.comparingDouble(Order::amount).reversed())
    .limit(3)
    .map(Order::customer)
    .toList();

// Result: [Alice, Charlie, Alice]
```

> Read it as: "From orders → keep completed → sort by amount descending → take top 3 → get names."
{: .prompt-tip }

---

## Working with Maps

### Iterate a Map

```java
Map<String, Integer> ages = Map.of("Anupam", 35, "Bob", 28, "Alice", 32);

ages.forEach((name, age) -> 
    System.out.println(name + " is " + age + " years old")
);
```

### Filter a Map

```java
Map<String, Integer> seniors = ages.entrySet().stream()
    .filter(e -> e.getValue() >= 30)
    .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));

// {Anupam=35, Alice=32}
```

---

## Useful Collection Factory Methods (Java 9+)

```java
// Immutable List
List<String> names = List.of("Anupam", "Bob", "Alice");

// Immutable Set
Set<Integer> numbers = Set.of(1, 2, 3, 4, 5);

// Immutable Map
Map<String, String> config = Map.of(
    "host", "localhost",
    "port", "8080",
    "env", "production"
);

// Map with more than 10 entries
Map<String, String> bigMap = Map.ofEntries(
    Map.entry("key1", "value1"),
    Map.entry("key2", "value2"),
    Map.entry("key3", "value3")
);
```

> These collections are **immutable** — you can't add or remove items. Use `new ArrayList<>(List.of(...))` if you need a mutable copy.
{: .prompt-warning }

---

## Quick Reference Cheat Sheet

| What You Want | Method | Example |
|---------------|--------|---------|
| Keep matching items | `.filter()` | `.filter(x -> x > 5)` |
| Transform items | `.map()` | `.map(String::toUpperCase)` |
| Sort | `.sorted()` | `.sorted(Comparator.comparing(...))` |
| Remove duplicates | `.distinct()` | `.distinct()` |
| Take first N | `.limit(n)` | `.limit(5)` |
| Skip first N | `.skip(n)` | `.skip(2)` |
| Find first match | `.findFirst()` | `.findFirst()` |
| Check all match | `.allMatch()` | `.allMatch(x -> x > 0)` |
| Check any match | `.anyMatch()` | `.anyMatch(x -> x > 100)` |
| Count items | `.count()` | `.count()` |
| Sum numbers | `.mapToInt().sum()` | `.mapToInt(x -> x).sum()` |
| Collect to List | `.toList()` | `.toList()` |
| Group by | `.collect(groupingBy())` | `.collect(Collectors.groupingBy(...))` |
| Join strings | `.collect(joining())` | `.collect(Collectors.joining(", "))` |
| To Map | `.collect(toMap())` | `.collect(Collectors.toMap(...))` |

---

## Common Mistakes to Avoid

### 1. Don't reuse a stream

```java
// This will throw IllegalStateException
Stream<String> stream = names.stream();
stream.filter(n -> n.startsWith("A")).toList();
stream.filter(n -> n.startsWith("B")).toList(); // BOOM!

// Create a new stream each time
names.stream().filter(n -> n.startsWith("A")).toList();
names.stream().filter(n -> n.startsWith("B")).toList();
```

### 2. Don't modify the source list inside a lambda

```java
// Bad — ConcurrentModificationException risk
names.stream().forEach(n -> names.remove(n));

// Good — collect to a new list
List<String> filtered = names.stream()
    .filter(n -> !n.startsWith("A"))
    .toList();
```

### 3. Use method references when possible

```java
// These are the same:
.map(name -> name.toUpperCase())
.map(String::toUpperCase)  // Cleaner

.forEach(item -> System.out.println(item))
.forEach(System.out::println)  // Cleaner
```

---

## When NOT to Use Streams

- **Simple loop with index** — if you need the index, a regular `for` loop is fine
- **Single operation** — `list.contains(x)` is better than `list.stream().anyMatch(...)`
- **Performance-critical tight loops** — streams have overhead; measure first
- **Complex mutable state** — if your logic builds complex state, a loop can be clearer

---

## The Pattern

The combination of Collections + Lambda follows one pattern:

```java
collection.stream()
    .filter(...)   // what to keep
    .map(...)      // how to transform
    .sorted(...)   // how to order
    .collect(...)  // what to produce
```

Start using these in your daily code. Once it clicks, you'll wonder how you ever wrote all those `for` loops.

---

## Further Reading

| Resource | Link |
|----------|------|
| Stream API — Oracle Docs | [Java SE 21 Stream](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/stream/Stream.html) |
| Collectors — Oracle Docs | [Java SE 21 Collectors](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/stream/Collectors.html) |
| Lambda Expressions — Oracle Tutorial | [Oracle Tutorial](https://docs.oracle.com/javase/tutorial/java/javaOO/lambdaexpressions.html) |
| Method References — Oracle Tutorial | [Oracle Tutorial](https://docs.oracle.com/javase/tutorial/java/javaOO/methodreferences.html) |
| Baeldung — Java Streams | [baeldung.com/java-streams](https://www.baeldung.com/java-streams) |
