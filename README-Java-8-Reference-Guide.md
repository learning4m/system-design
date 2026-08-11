# Java 8 — Streams, Collectors, Reduce & Functional Programming Reference

> Consolidated reference from the Java 8 project discussions. The goal is to preserve the *reasoning* behind the APIs, not just syntax.

## Table of Contents

1. [Stream Mental Model](#1-stream-mental-model)
2. [map vs flatMap](#2-map-vs-flatmap)
3. [Stream.of vs Arrays.stream](#3-streamof-vs-arraysstream)
4. [Primitive Streams](#4-primitive-streams)
5. [map, mapToInt and mapToObj](#5-map-maptoint-and-maptoobj)
6. [reduce — All Three Forms](#6-reduce--all-three-forms)
7. [Implementing sum, min, max and average with reduce](#7-implementing-sum-min-max-and-average-with-reduce)
8. [reduce vs collect](#8-reduce-vs-collect)
9. [Collectors.reducing](#9-collectorsreducing)
10. [Comparator: compare, comparing and primitive variants](#10-comparator-compare-comparing-and-primitive-variants)
11. [max/min vs maxBy/minBy](#11-maxmin-vs-maxbyminby)
12. [groupingBy and Downstream Collectors](#12-groupingby-and-downstream-collectors)
13. [mapping](#13-mapping)
14. [collectingAndThen](#14-collectingandthen)
15. [Finding Duplicate Employees](#15-finding-duplicate-employees)
16. [Removing Duplicates](#16-removing-duplicates)
17. [Finding Duplicate Characters in a String](#17-finding-duplicate-characters-in-a-string)
18. [String.chars and Character Streams](#18-stringchars-and-character-streams)
19. [Parallel Streams](#19-parallel-streams)
20. [Quick Decision Guide](#20-quick-decision-guide)

---

## 1. Stream Mental Model

A stream is a pipeline over data:

```text
Source
  |
  v
Intermediate operation
  |
  v
Intermediate operation
  |
  v
Terminal operation
```

Example:

```java
List<String> names = employees.stream()
        .filter(e -> e.getSalary() > 100000)
        .map(Employee::getName)
        .sorted()
        .collect(Collectors.toList());
```

Think of the types flowing through the pipeline:

```text
List<Employee>
     |
   stream()
     v
Stream<Employee>
     |
   filter(...)
     v
Stream<Employee>
     |
   map(Employee::getName)
     v
Stream<String>
     |
   sorted()
     v
Stream<String>
     |
   collect(...)
     v
List<String>
```

This type-flow view is especially useful for understanding `map`, `flatMap`, primitive streams and collectors.

---

## 2. map vs flatMap

### map

`map` transforms **one element into one element**.

```java
<R> Stream<R> map(Function<T, R> mapper)
```

Example:

```java
List<String> words = Arrays.asList("JAVA", "STREAM");

Stream<Integer> lengths = words.stream()
        .map(String::length);
```

Flow:

```text
"JAVA"   -> 4
"STREAM" -> 6
```

### When the mapper returns a collection/array/stream

```java
words.stream()
     .map(word -> word.split(""));
```

`split` returns `String[]`, therefore:

```text
Stream<String>
     |
     | map(word -> word.split(""))
     v
Stream<String[]>
```

That is a stream whose **elements are arrays**.

### flatMap

`flatMap` maps each input to a stream and then flattens all those inner streams.

```java
<R> Stream<R> flatMap(Function<T, ? extends Stream<? extends R>> mapper)
```

Instead of:

```java
words.stream()
     .map(word -> word.split(""))
     .flatMap(Arrays::stream)
```

you can directly write:

```java
List<String> characters = words.stream()
        .flatMap(word -> Arrays.stream(word.split("")))
        .collect(Collectors.toList());
```

Flow:

```text
Stream<String>
     |
     | flatMap
     v
word 1 -> Stream<String> --\
word 2 -> Stream<String> ----> Stream<String>
word 3 -> Stream<String> --/
```

### Core rule

```text
map     : T -> R
flatMap : T -> Stream<R>
```

Use `flatMap` when one source element can produce **zero, one or many output elements**.

---

## 3. Stream.of vs Arrays.stream

For an object array:

```java
String[] values = {"A", "B", "C"};
```

both work:

```java
Arrays.stream(values)
Stream.of(values)
```

Both produce:

```text
Stream<String>
```

So this is valid:

```java
words.stream()
     .flatMap(word -> Stream.of(word.split("")))
```

and is equivalent in this context to:

```java
words.stream()
     .flatMap(word -> Arrays.stream(word.split("")))
```

### Primitive-array caveat

Primitive arrays behave differently with `Stream.of`.

```java
int[] numbers = {1, 2, 3};
```

```java
Arrays.stream(numbers)
```

produces:

```text
IntStream
```

whereas:

```java
Stream.of(numbers)
```

produces effectively:

```text
Stream<int[]>
```

because the entire `int[]` is treated as one object.

---

## 4. Primitive Streams

Java provides three specialized primitive streams:

```text
IntStream
LongStream
DoubleStream
```

They avoid unnecessary boxing and expose numeric operations directly.

### Common numeric operations

`IntStream` supports operations such as:

```java
sum()
average()
min()
max()
summaryStatistics()
range()
rangeClosed()
```

Example:

```java
int total = employees.stream()
        .mapToInt(Employee::getSalary)
        .sum();
```

Without a primitive stream you may write:

```java
int total = employees.stream()
        .map(Employee::getSalary)
        .reduce(0, Integer::sum);
```

The primitive form is normally preferable for numeric calculations.

### IntStream-specific creation methods

```java
IntStream.range(1, 5);       // 1,2,3,4
IntStream.rangeClosed(1, 5); // 1,2,3,4,5
```

Equivalent methods exist for `LongStream`. `DoubleStream` does not have `range/rangeClosed` because ranges of floating-point values have problematic semantics.

### Conversions

```text
Stream<T> --mapToInt--> IntStream
Stream<T> --mapToLong-> LongStream
Stream<T> --mapToDouble-> DoubleStream

IntStream --boxed()----> Stream<Integer>
IntStream --mapToObj()--> Stream<R>
```

---

## 5. map, mapToInt and mapToObj

### Stream.map

Object -> Object

```java
employees.stream()
         .map(Employee::getName)
```

```text
Stream<Employee> -> Stream<String>
```

### Stream.mapToInt

Object -> primitive int

```java
employees.stream()
         .mapToInt(Employee::getSalary)
```

```text
Stream<Employee> -> IntStream
```

This unlocks:

```java
.sum()
.average()
.summaryStatistics()
```

### IntStream.map

Primitive int -> primitive int

```java
IntStream.rangeClosed(1, 5)
         .map(n -> n * n);
```

Still returns:

```text
IntStream
```

### IntStream.mapToObj

Primitive int -> Object

```java
IntStream.rangeClosed(1, 5)
         .mapToObj(n -> "Number-" + n);
```

returns:

```text
Stream<String>
```

### Mental model

```text
map       = remain in the same general stream family
mapToInt  = object stream -> primitive stream
mapToObj  = primitive stream -> object stream
```

---

## 6. reduce — All Three Forms

Reduction combines many stream elements into one result.

Conceptually:

```text
[a, b, c, d]
     |
     v
combine repeatedly
     |
     v
single result
```

### Form 1 — accumulator only

```java
Optional<T> reduce(BinaryOperator<T> accumulator)
```

Example:

```java
Optional<Integer> sum = numbers.stream()
        .reduce(Integer::sum);
```

Why `Optional`?

Because the stream may be empty and there is no supplied identity value.

```text
[10, 20, 30]
10 + 20 = 30
30 + 30 = 60
=> Optional[60]
```

### Form 2 — identity + accumulator

```java
T reduce(T identity, BinaryOperator<T> accumulator)
```

Example:

```java
int sum = numbers.stream()
        .reduce(0, Integer::sum);
```

`0` is the identity for addition because:

```text
0 + x = x
```

For multiplication:

```java
int product = numbers.stream()
        .reduce(1, (a, b) -> a * b);
```

`1` is the identity because:

```text
1 * x = x
```

### Form 3 — identity + accumulator + combiner

```java
<U> U reduce(
    U identity,
    BiFunction<U, ? super T, U> accumulator,
    BinaryOperator<U> combiner)
```

This form allows the result type to differ from the stream element type.

Example:

```java
int totalLength = words.parallelStream()
        .reduce(
            0,
            (length, word) -> length + word.length(),
            Integer::sum
        );
```

Types:

```text
Stream element = String
Result         = Integer
```

Parallel conceptual flow:

```text
Partition 1                 Partition 2
----------                  ----------
"Java" -> 4                "Streams" -> 7
"API"  -> +3               "Reduce"  -> +6
          |                             |
          v                             v
          7                            13
           \                           /
            \------ combiner --------/
                       |
                       v
                      20
```

### Important reduction rules

The operation should be associative:

```text
(a op b) op c == a op (b op c)
```

This matters especially for parallel streams.

Do not use mutable identity containers with `reduce`, such as:

```java
new ArrayList<>()
```

and mutate them inside the accumulator. `collect` is designed for mutable accumulation.

---

## 7. Implementing sum, min, max and average with reduce

### Sum

```java
int sum = numbers.stream()
        .reduce(0, Integer::sum);
```

Equivalent primitive operation:

```java
numbers.stream()
       .mapToInt(Integer::intValue)
       .sum();
```

### Maximum

```java
Optional<Integer> max = numbers.stream()
        .reduce(Integer::max);
```

or:

```java
Optional<Integer> max = numbers.stream()
        .reduce((a, b) -> a > b ? a : b);
```

Equivalent:

```java
numbers.stream().max(Integer::compareTo);
```

### Minimum

```java
Optional<Integer> min = numbers.stream()
        .reduce(Integer::min);
```

### Average

Average is not naturally a simple scalar reduction because you need both:

```text
sum
count
```

A custom mutable/immutable aggregate could track both, but for normal code prefer:

```java
OptionalDouble avg = numbers.stream()
        .mapToInt(Integer::intValue)
        .average();
```

or a summary collector/statistics object.

---

## 8. reduce vs collect

Both aggregate a stream, but they solve different kinds of aggregation.

### reduce

Best for immutable value reduction:

```text
many values -> one value
```

Examples:

```text
numbers -> sum
numbers -> maximum
strings -> concatenated string
```

### collect

Best for mutable result containers:

```text
many values -> mutable container/result structure
```

Examples:

```text
Stream<Employee> -> List<Employee>
Stream<Employee> -> Set<Employee>
Stream<Employee> -> Map<Department,List<Employee>>
```

Example:

```java
List<Employee> list = employees.stream()
        .collect(Collectors.toList());
```

### Why not reduce into ArrayList?

Avoid patterns like:

```java
employees.stream().reduce(
    new ArrayList<>(),
    (list, employee) -> {
        list.add(employee);
        return list;
    },
    ...
);
```

The identity is mutable, which breaks assumptions behind reduction and can behave badly in parallel execution.

Use:

```java
employees.stream().collect(Collectors.toList());
```

### Rule

```text
Immutable scalar/value aggregation -> reduce
Mutable accumulation              -> collect
```

---

## 9. Collectors.reducing

There are two related APIs:

```java
Stream.reduce(...)
Collectors.reducing(...)
```

Why have a reducing collector?

Because downstream operations of collectors such as `groupingBy` expect another `Collector`.

Example:

```java
Map<String, Optional<Employee>> highestPaidByDepartment =
    employees.stream()
        .collect(Collectors.groupingBy(
            Employee::getDepartment,
            Collectors.reducing(
                BinaryOperator.maxBy(
                    Comparator.comparing(Employee::getSalary)
                )
            )
        ));
```

Conceptually:

```text
groupingBy
    |
    +-- IT ----> reducing(...)
    |
    +-- HR ----> reducing(...)
    |
    +-- FIN ---> reducing(...)
```

You cannot directly place `Stream.reduce` there because `groupingBy` needs a `Collector`, not a terminal stream operation.

---

## 10. Comparator: compare, comparing and primitive variants

### compare

`compare` actually compares **two objects**.

```java
Comparator<Employee> comparator =
        (e1, e2) -> e1.getName().compareTo(e2.getName());

int result = comparator.compare(emp1, emp2);
```

The result is:

```text
negative -> first < second
zero     -> equal for ordering
positive -> first > second
```

### Comparator.comparing

`comparing` **builds a Comparator** from a key extractor.

```java
Comparator<Employee> byName =
        Comparator.comparing(Employee::getName);
```

Conceptually:

```text
Employee
   |
   | getName
   v
String
   |
   | compareTo
   v
comparison result
```

### comparingInt / comparingLong / comparingDouble

Example:

```java
Comparator<Employee> byAge =
        Comparator.comparingInt(Employee::getAge);
```

You could write:

```java
Comparator.comparing(Employee::getAge)
```

if `getAge()` results in an `Integer`, but this can involve boxing.

Primitive variants avoid boxing:

```text
comparingInt
comparingLong
comparingDouble
```

Prefer them for primitive numeric keys.

### Replacing comparingInt manually

```java
Comparator<Employee> byAge =
        (e1, e2) -> Integer.compare(e1.getAge(), e2.getAge());
```

`Comparator.comparingInt` is simply clearer and reusable.

---

## 11. max/min vs maxBy/minBy

### Stream.max

`Stream.max` is a **terminal stream operation**.

```java
Optional<Employee> highestPaid = employees.stream()
        .max(Comparator.comparingInt(Employee::getSalary));
```

Signature conceptually:

```java
Optional<T> max(Comparator<? super T> comparator)
```

There is no no-argument `Stream.max()` for object streams because Java does not assume every `T` is naturally comparable.

Primitive streams are different:

```java
IntStream.of(1, 5, 3).max();
```

No comparator is needed because ints have a defined numeric ordering.

### BinaryOperator.maxBy

`BinaryOperator.maxBy` creates a **BinaryOperator** that selects the larger of two values according to a comparator.

```java
BinaryOperator<Employee> operator =
        BinaryOperator.maxBy(
            Comparator.comparingInt(Employee::getSalary)
        );
```

Useful with `reduce`:

```java
Optional<Employee> max = employees.stream()
        .reduce(BinaryOperator.maxBy(
            Comparator.comparingInt(Employee::getSalary)
        ));
```

### Collectors.maxBy

`Collectors.maxBy` returns a **Collector**.

```java
Optional<Employee> max = employees.stream()
        .collect(Collectors.maxBy(
            Comparator.comparingInt(Employee::getSalary)
        ));
```

Its major value appears as a downstream collector:

```java
Map<String, Optional<Employee>> result = employees.stream()
        .collect(Collectors.groupingBy(
            Employee::getDepartment,
            Collectors.maxBy(
                Comparator.comparingInt(Employee::getSalary)
            )
        ));
```

### Comparison

| API | Produces/Returns | Typical role |
|---|---|---|
| `stream.max(cmp)` | `Optional<T>` | terminal stream operation |
| `BinaryOperator.maxBy(cmp)` | `BinaryOperator<T>` | reduction operator |
| `Collectors.maxBy(cmp)` | `Collector<T,...,Optional<T>>` | collector/downstream collector |

---

## 12. groupingBy and Downstream Collectors

Basic grouping:

```java
Map<String, List<Employee>> result = employees.stream()
        .collect(Collectors.groupingBy(Employee::getDepartment));
```

Conceptually:

```text
Employees
   |
   | department classifier
   v
+-----------------------+
| IT  -> [e1,e2,e3]     |
| HR  -> [e4,e5]        |
| FIN -> [e6,e7]        |
+-----------------------+
```

`groupingBy` can accept a downstream collector:

```java
groupingBy(classifier, downstreamCollector)
```

Example:

```java
Map<String, Long> countByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
            Employee::getDepartment,
            Collectors.counting()
        ));
```

Now the downstream collector transforms each group's contents:

```text
IT  -> counting() -> 3
HR  -> counting() -> 2
FIN -> counting() -> 2
```

Other downstream collectors include:

```text
mapping
reducing
maxBy
minBy
summingInt
averagingInt
counting
collectingAndThen
```

---

## 13. mapping

`Collectors.mapping` is the collector-world counterpart to `Stream.map`.

Example:

```java
Map<String, List<String>> employeeNamesByDepartment =
    employees.stream()
        .collect(Collectors.groupingBy(
            Employee::getDepartment,
            Collectors.mapping(
                Employee::getName,
                Collectors.toList()
            )
        ));
```

Flow:

```text
Employee
   |
   | groupingBy department
   v
Department group
   |
   | mapping Employee -> name
   v
String
   |
   | toList
   v
List<String>
```

Why not simply `map` before `groupingBy`?

Because if you map `Employee -> String` before grouping, you may lose the employee information needed by the classifier.

Downstream `mapping` lets you:

```text
first classify Employee
then transform elements inside each group
```

---

## 14. collectingAndThen

`collectingAndThen` performs a final transformation on the result produced by another collector.

Signature conceptually:

```java
collectingAndThen(
    Collector<T, A, R> downstream,
    Function<R, RR> finisher
)
```

Notice the order:

```text
Collector first
Function second
```

Example:

```java
List<String> immutableNames = employees.stream()
        .map(Employee::getName)
        .collect(Collectors.collectingAndThen(
            Collectors.toList(),
            Collections::unmodifiableList
        ));
```

Flow:

```text
Stream
  |
  | toList()
  v
List
  |
  | finisher
  v
Unmodifiable List
```

### With groupingBy

```java
Map<String, Integer> countByDepartment = employees.stream()
        .collect(Collectors.groupingBy(
            Employee::getDepartment,
            Collectors.collectingAndThen(
                Collectors.toList(),
                List::size
            )
        ));
```

Here `collectingAndThen` operates on each group's downstream result because it is nested inside `groupingBy`.

But it is **not inherently limited to map values**. It can be used as a top-level collector too.

### Compare with mapping

```text
mapping
-------
transforms each ELEMENT before downstream collection

collectingAndThen
-----------------
transforms the FINAL RESULT of a collector
```

Example:

```text
mapping:
Employee -> name -> collect names

collectingAndThen:
Employees -> collect List<Employee> -> List.size()
```

---

## 15. Finding Duplicate Employees

Suppose duplicates are defined by the combination:

```text
name + department
```

### groupingBy approach

```java
Map<String, Long> counts = employees.stream()
        .collect(Collectors.groupingBy(
            e -> e.getName() + "|" + e.getDepartment(),
            Collectors.counting()
        ));

List<String> duplicates = counts.entrySet().stream()
        .filter(e -> e.getValue() > 1)
        .map(Map.Entry::getKey)
        .collect(Collectors.toList());
```

A better production design is to use a proper key object instead of string concatenation.

### Without groupingBy

A stateful set-based approach can detect repeated keys:

```java
Set<String> seen = new HashSet<>();

List<Employee> duplicates = employees.stream()
        .filter(e -> !seen.add(e.getName() + "|" + e.getDepartment()))
        .collect(Collectors.toList());
```

Why does it work?

```java
seen.add(key)
```

returns:

```text
true  -> first occurrence
false -> key already existed
```

So:

```java
!seen.add(key)
```

selects later duplicate occurrences.

Caution: this introduces external mutable state and is not appropriate for a parallel stream.

---

## 16. Removing Duplicates

### distinct

```java
List<Employee> unique = employees.stream()
        .distinct()
        .collect(Collectors.toList());
```

`distinct()` relies on:

```text
equals()
hashCode()
```

Therefore, if two employees should be equal when name and department match, `Employee.equals/hashCode` must represent that definition.

### Collectors.toSet

```java
Set<Employee> unique = employees.stream()
        .collect(Collectors.toSet());
```

This also uses equality/hash semantics to eliminate duplicates.

Difference in intent/result:

```text
distinct().collect(toList()) -> List, stream order can be preserved
collect(toSet())             -> Set, set semantics; default collector does not promise encounter order
```

If you specifically need a set with encounter order:

```java
.collect(Collectors.toCollection(LinkedHashSet::new))
```

### Duplicate by selected fields only

If object equality is not defined by the fields you need, group by or maintain a set of a derived key rather than changing global equality just for one query.

---

## 17. Finding Duplicate Characters in a String

Example:

```text
"programming"
```

One approach:

```java
Map<Character, Long> frequency = input.chars()
        .mapToObj(c -> (char) c)
        .collect(Collectors.groupingBy(
            Function.identity(),
            Collectors.counting()
        ));

List<Character> duplicates = frequency.entrySet().stream()
        .filter(e -> e.getValue() > 1)
        .map(Map.Entry::getKey)
        .collect(Collectors.toList());
```

### Without groupingBy

```java
Set<Character> seen = new HashSet<>();
Set<Character> duplicates = input.chars()
        .mapToObj(c -> (char) c)
        .filter(c -> !seen.add(c))
        .collect(Collectors.toSet());
```

Again, this stateful approach should remain sequential.

---

## 18. String.chars and Character Streams

### What does String.chars() return?

```java
IntStream chars()
```

Example:

```java
"ABC".chars()
```

conceptually produces integer values corresponding to UTF-16 code units:

```text
65, 66, 67
```

To convert to characters:

```java
Stream<Character> characters = input.chars()
        .mapToObj(c -> (char) c);
```

### Why not Arrays.stream(string.toCharArray())?

Because Java has no:

```java
Arrays.stream(char[])
```

overload.

`Arrays.stream` supports primitive arrays such as:

```text
int[]
long[]
double[]
```

but not `char[]`.

### Character[] works

If you have:

```java
Character[] chars;
```

then:

```java
Arrays.stream(chars)
```

returns:

```text
Stream<Character>
```

### Convert String to Character[]

One Java 8 approach:

```java
Character[] chars = input.chars()
        .mapToObj(c -> (char) c)
        .toArray(Character[]::new);
```

But if your only goal is stream processing, creating the intermediate `Character[]` is usually unnecessary.

### Unicode caveat

`String.chars()` operates on UTF-16 code units. For full Unicode code points, use:

```java
input.codePoints()
```

which also returns an `IntStream`.

---

## 19. Parallel Streams

A parallel stream divides work among threads, commonly using the common `ForkJoinPool`.

```java
list.parallelStream()
    .map(...)
    .filter(...)
    .collect(...);
```

Conceptually:

```text
                  Input
                    |
          +---------+---------+
          |                   |
      partition A         partition B
          |                   |
      worker-1             worker-2
          |                   |
          +---------+---------+
                    |
                 combine
                    |
                  result
```

### Parallel does not mean every operation is safe

Operations should avoid shared mutable state.

Bad:

```java
List<Integer> result = new ArrayList<>();

numbers.parallelStream()
       .forEach(result::add);
```

Multiple workers may mutate the same `ArrayList` concurrently.

Prefer:

```java
List<Integer> result = numbers.parallelStream()
        .collect(Collectors.toList());
```

### Reduction must be associative

Addition:

```text
(a + b) + c == a + (b + c)
```

Subtraction:

```text
(a - b) - c != a - (b - c)
```

Therefore:

```java
parallelStream().reduce(0, (a, b) -> a - b)
```

is not a valid parallel reduction strategy.

### When parallel streams may help

Typical favorable conditions:

```text
large data set
CPU-intensive work
independent elements
associative reduction
little shared state
splittable source such as ArrayList/array
```

### When to avoid them

```text
small collections
blocking I/O
shared mutable state
strict thread-affinity requirements
non-associative reductions
code running in an environment where common-pool usage is undesirable
```

Do not assume `parallelStream()` automatically makes code faster. Splitting, scheduling, coordination and combining have overhead.

---

## 20. Quick Decision Guide

### Transform one element to one element

```java
map(...)
```

### Transform one element into many and flatten

```java
flatMap(...)
```

### Need numeric stream operations

```java
mapToInt(...)
mapToLong(...)
mapToDouble(...)
```

### Primitive stream back to objects

```java
mapToObj(...)
boxed()
```

### Combine values into one immutable value

```java
reduce(...)
```

### Build a List/Set/Map or other mutable result

```java
collect(...)
```

### Group elements

```java
Collectors.groupingBy(...)
```

### Transform each element inside a group

```java
Collectors.mapping(...)
```

### Transform the final result of another collector

```java
Collectors.collectingAndThen(...)
```

### Maximum directly on a stream

```java
stream.max(comparator)
```

### Maximum as a reduction operator

```java
BinaryOperator.maxBy(comparator)
```

### Maximum inside groupingBy

```java
Collectors.maxBy(comparator)
```

### Numeric aggregation inside groupingBy

```java
Collectors.summingInt(...)
Collectors.averagingInt(...)
Collectors.summarizingInt(...)
```

### Remove duplicates according to equals/hashCode

```java
distinct()
```

or:

```java
Collectors.toSet()
```

---

# Final Mental Model

The most useful way to organize the Java 8 Stream API is by **what kind of transformation is happening**:

```text
                         STREAM PIPELINE
                               |
        +----------------------+----------------------+
        |                      |                      |
   Transformation         Aggregation           Organization
        |                      |                      |
   map / flatMap          reduce / collect       groupingBy
   mapToInt               min / max              partitioningBy
   mapToObj               sum / average          toMap
                               |
                    +----------+----------+
                    |                     |
              immutable result      mutable result
                    |                     |
                  reduce                collect
```

And for collectors:

```text
                    Collector
                       |
        +--------------+---------------+
        |              |               |
   Build result     Aggregate       Adapt result
        |              |               |
   toList          counting         mapping
   toSet           summingInt       collectingAndThen
   toMap           maxBy            reducing
                   minBy
```

The central distinction to remember is:

```text
Stream operation     -> operates on the stream pipeline
Collector            -> describes how collect() should accumulate data
Downstream collector -> describes what to do inside each group/partition
```

Once that distinction is clear, APIs such as `reduce` vs `Collectors.reducing`, `max` vs `maxBy`, and `map` vs `Collectors.mapping` become much easier to reason about.
