---
title: java8 stream应用
date: 2018-07-17 17:05:57
tags: [java8,stream]
categories: [java]
---

# stream的生成
### 从 Collection 和数组

```
Collection.stream()
Collection.parallelStream()
Arrays.stream(T array) 
Stream.of()
```
### 从 BufferedReader

```
java.io.BufferedReader.lines()
```
### 静态工厂

```
java.util.stream.IntStream.range()
java.nio.file.Files.walk()
```
### 其它

```
Random.ints()
BitSet.stream()
Pattern.splitAsStream(java.lang.CharSequence)
JarFile.stream()
```

### Stream.generate
- 生成 10 个随机整数
```
Random seed = new Random();
Supplier<Integer> random = seed::nextInt;
Stream.generate(random).limit(10).forEach(System.out::println);
//Another way
IntStream.generate(() -> (int) (System.nanoTime() % 100)).
limit(10).forEach(System.out::println);
```
### Stream.iterate
iterate 跟 reduce 操作很像，接受一个种子值，和一个 UnaryOperator（例如 f）。然后种子值成为 Stream 的第一个元素，f(seed) 为第二个，f(f(seed)) 第三个，以此类推。
- 生成一个等差数列
```
Stream.iterate(0, n -> n + 3).limit(10). forEach(x -> System.out.print(x + " "));
```


# 流的使用详解
> 简单说，对 Stream 的使用就是实现一个 filter-map-reduce 过程，产生一个最终结果，或者导致一个副作用

**当把一个数据结构包装成Stream 后，就要开始对里面的元素进行各类操作了。常见的操作可以归类如下。**

- Intermediate：

> map (mapToInt, flatMap 等)、 filter、 distinct、 sorted、 peek、 limit、 skip、 parallel、 sequential、 unordered

- Terminal：

> forEach、 forEachOrdered、 toArray、 reduce、 collect、 min、 max、 count、 anyMatch、 allMatch、 noneMatch、 findFirst、 findAny、 iterator

- Short-circuiting：

> anyMatch、 allMatch、 noneMatch、 findFirst、 findAny、 limit

### map/flatMap
作用就是把 input Stream 的每一个元素，映射成 output Stream 的另外一个元素。
- 转换大写
```
List<String> output = wordList.stream().
map(String::toUpperCase).
collect(Collectors.toList());
```

- 平方数
```
List<Integer> nums = Arrays.asList(1, 2, 3, 4);
List<Integer> squareNums = nums.stream().
map(n -> n * n).
collect(Collectors.toList());
```
- 一对多
```
Stream<List<Integer>> inputStream = Stream.of(
 Arrays.asList(1),
 Arrays.asList(2, 3),
 Arrays.asList(4, 5, 6)
 );
Stream<Integer> outputStream = inputStream.
flatMap((childList) -> childList.stream());
```
### filter
filter 对原始 Stream 进行某项测试，通过测试的元素被留下来生成一个新 Stream。

- 留下偶数
```
Integer[] sixNums = {1, 2, 3, 4, 5, 6};
Integer[] evens =
Stream.of(sixNums).filter(n -> n%2 == 0).toArray(Integer[]::new);
```

- 把单词挑出来
```
List<String> output = reader.lines().
 flatMap(line -> Stream.of(line.split(REGEXP))).
 filter(word -> word.length() > 0).
 collect(Collectors.toList());
```

### forEach
forEach 方法接收一个 Lambda 表达式，然后在 Stream 的每一个元素上执行该表达式。
-  打印姓名（forEach 和 pre-java8 的对比）
```
// Java 8
roster.stream()
 .filter(p -> p.getGender() == Person.Sex.MALE)
 .forEach(p -> System.out.println(p.getName()));
// Pre-Java 8
for (Person p : roster) {
 if (p.getGender() == Person.Sex.MALE) {
 System.out.println(p.getName());
 }
}
```
### findFirst
这是一个 termimal 兼 short-circuiting 操作，它总是返回 Stream 的第一个元素，或者空。这里比较重点的是它的返回值类型：Optional。这也是一个模仿 Scala 语言中的概念，作为一个容器，它可能含有某值，或者不包含。使用它的目的是尽可能避免 NullPointerException。

```
String strA = " abcd ", strB = null;
print(strA);
print("");
print(strB);
getLength(strA);
getLength("");
getLength(strB);
public static void print(String text) {
 // Java 8
 Optional.ofNullable(text).ifPresent(System.out::println);
 // Pre-Java 8
 if (text != null) {
 System.out.println(text);
 }
 }
public static int getLength(String text) {
 // Java 8
return Optional.ofNullable(text).map(String::length).orElse(-1);
 // Pre-Java 8
// return if (text != null) ? text.length() : -1;
 };
```
> 在更复杂的 if (xx != null) 的情况中，使用 Optional 代码的可读性更好，而且它提供的是编译时检查，能极大的降低 NPE 这种 Runtime Exception 对程序的影响，或者迫使程序员更早的在编码阶段处理空值问题，而不是留到运行时再发现和调试。Stream 中的 findAny、max/min、reduce 等方法等返回 Optional 值。还有例如 IntStream.average() 返回 OptionalDouble 等等。

### reduce
> T reduce(T identity, BinaryOperator<T> accumulator) 这个方法的主要作用是把 Stream 元素组合起来。它提供一个起始值（种子），然后依照运算规则（BinaryOperator），和前面 Stream 的第一个、第二个、第 n 个元素组合。从这个意义上说，字符串拼接、数值的 sum、min、max、average 都是特殊的 reduce.也有没有起始值的情况，这时会把 Stream 的前面两个元素组合起来，返回的是 Optional。

```
// 字符串连接，concat = "ABCD"
String concat = Stream.of("A", "B", "C", "D").reduce("", String::concat); 
// 求最小值，minValue = -3.0
double minValue = Stream.of(-1.5, 1.0, -3.0, -2.0).reduce(Double.MAX_VALUE, Double::min); 
// 求和，sumValue = 10, 有起始值
int sumValue = Stream.of(1, 2, 3, 4).reduce(0, Integer::sum);
// 求和，sumValue = 10, 无起始值
sumValue = Stream.of(1, 2, 3, 4).reduce(Integer::sum).get();
// 过滤，字符串连接，concat = "ace"
concat = Stream.of("a", "B", "c", "D", "e", "F").
 filter(x -> x.compareTo("Z") > 0).
 reduce("", String::concat);
```
### limit/skip
> limit 返回 Stream 的前面 n 个元素；skip 则是扔掉前 n 个元素（它是由一个叫 subStream 的方法改名而来）。

```
public void testLimitAndSkip() {
 List<Person> persons = new ArrayList();
 for (int i = 1; i <= 10000; i++) {
 Person person = new Person(i, "name" + i);
 persons.add(person);
 }
List<String> personList2 = persons.stream().
map(Person::getName).limit(10).skip(3).collect(Collectors.toList());
 System.out.println(personList2);
}
private class Person {
 public int no;
 private String name;
 public Person (int no, String name) {
 this.no = no;
 this.name = name;
 }
 public String getName() {
 System.out.println(name);
 return name;
 }
}
```

### sorted
> 对 Stream 的排序通过 sorted 进行，它比数组的排序更强之处在于你可以首先对 Stream 进行各类 map、filter、limit、skip 甚至 distinct 来减少元素数量后，再排序，这能帮助程序明显缩短执行时间。

- 排序前进行 limit 和 skip
```
List<Person> persons = new ArrayList();
 for (int i = 1; i <= 5; i++) {
 Person person = new Person(i, "name" + i);
 persons.add(person);
 }
List<Person> personList2 = persons.stream().limit(2).sorted((p1, p2) -> p1.getName().compareTo(p2.getName())).collect(Collectors.toList());
System.out.println(personList2);
```
### min/max/distinct
> min 和 max 的功能也可以通过对 Stream 元素先排序，再 findFirst 来实现，但前者的性能会更好，为 O(n)，而 sorted 的成本是 O(n log n)。同时它们作为特殊的 reduce 方法被独立出来也是因为求最大最小值是很常见的操作。

-  找出最长一行的长度
```
BufferedReader br = new BufferedReader(new FileReader("c:\\SUService.log"));
int longest = br.lines().
 mapToInt(String::length).
 max().
 getAsInt();
br.close();
System.out.println(longest);
```
- 找出全文的单词，转小写，并排序

```
List<String> words = br.lines().
 flatMap(line -> Stream.of(line.split(" "))).
 filter(word -> word.length() > 0).
 map(String::toLowerCase).
 distinct().
 sorted().
 collect(Collectors.toList());
br.close();
System.out.println(words);
```
### Match
Stream 有三个 match 方法，从语义上说：

    allMatch：Stream 中全部元素符合传入的 predicate，返回 true
    anyMatch：Stream 中只要有一个元素符合传入的 predicate，返回 true
    noneMatch：Stream 中没有一个元素符合传入的 predicate，返回 true
    
> 它们都不是要遍历全部元素才能返回结果。例如 allMatch 只要一个元素不满足条件，就 skip 剩下的所有元素，返回 false。

```
List<Person> persons = new ArrayList();
persons.add(new Person(1, "name" + 1, 10));
persons.add(new Person(2, "name" + 2, 21));
persons.add(new Person(3, "name" + 3, 34));
persons.add(new Person(4, "name" + 4, 6));
persons.add(new Person(5, "name" + 5, 55));
boolean isAllAdult = persons.stream().
 allMatch(p -> p.getAge() > 18);
System.out.println("All are adult? " + isAllAdult);
boolean isThereAnyChild = persons.stream().
 anyMatch(p -> p.getAge() < 12);
System.out.println("Any child? " + isThereAnyChild);
```

## 用 Collectors 来进行 reduction 操作
> java.util.stream.Collectors 类的主要作用就是辅助进行各类有用的 reduction 操作，例如转变输出为 Collection，把 Stream 元素进行归组。

- 收集新的List
```
List<Integer> list = Stream.of(1, 2, 3, 4).filter(p -> p > 2).collect(Collectors.toList());
```
### groupingBy/partitioningBy

- 按照年龄归组
```
Map<Integer, List<Person>> personGroups = Stream.generate(new PersonSupplier()).
 limit(100).
 collect(Collectors.groupingBy(Person::getAge));
Iterator it = personGroups.entrySet().iterator();
while (it.hasNext()) {
 Map.Entry<Integer, List<Person>> persons = (Map.Entry) it.next();
 System.out.println("Age " + persons.getKey() + " = " + persons.getValue().size());
}
```

## reduce 与 collect区别
>Stream.reduce，常用的方法有average, sum, min, max, count，返回单个的结果值，并且reduce操作每处理一个元素总是创建一个新值
>Stream.collection与stream.reduce方法不同，Stream.collect修改现存的值，而不是每处理一个元素，创建一个新值

```
public class LambdaMapReduce { 
    private static List<User> users = Arrays.asList( 
            new User(1, "张三", 12,User.Sex.MALE), 
            new User(2, "李四", 21, User.Sex.FEMALE), 
            new User(3,"王五", 32, User.Sex.MALE), 
            new User(4, "赵六", 32, User.Sex.FEMALE)); 
  
    public static void main(String[] args) { 
        reduceAvg(); 
        reduceSum(); 
  
          
        //与stream.reduce方法不同，Stream.collect修改现存的值，而不是每处理一个元素，创建一个新值 
        //获取所有男性用户的平均年龄 
        Averager averageCollect = users.parallelStream() 
                .filter(p -> p.getGender() == User.Sex.MALE) 
                .map(User::getAge) 
                .collect(Averager::new, Averager::accept, Averager::combine); 
  
        System.out.println("Average age of male members: " 
                + averageCollect.average()); 
  
        //获取年龄大于12的用户列表 
        List<User> list = users.parallelStream().filter(p -> p.age > 12) 
                .collect(Collectors.toList()); 
        System.out.println(list); 
  
        //按性别统计用户数 
        Map<User.Sex, Integer> map = users.parallelStream().collect( 
                Collectors.groupingBy(User::getGender, 
                        Collectors.summingInt(p -> 1))); 
        System.out.println(map); 
  
        //按性别获取用户名称 
        Map<User.Sex, List<String>> map2 = users.stream() 
                .collect( 
                        Collectors.groupingBy( 
                                User::getGender, 
                                Collectors.mapping(User::getName, 
                                        Collectors.toList()))); 
        System.out.println(map2); 
          
        //按性别求年龄的总和 
        Map<User.Sex, Integer> map3 = users.stream().collect( 
                Collectors.groupingBy(User::getGender, 
                        Collectors.reducing(0, User::getAge, Integer::sum))); 
  
        System.out.println(map3); 
          
        //按性别求年龄的平均值 
        Map<User.Sex, Double> map4 = users.stream().collect( 
                Collectors.groupingBy(User::getGender, 
                        Collectors.averagingInt(User::getAge))); 
        System.out.println(map4); 
  
    } 
  
    // 注意，reduce操作每处理一个元素总是创建一个新值， 
    // Stream.reduce适用于返回单个结果值的情况 
    //获取所有用户的平均年龄 
    private static void reduceAvg() { 
        // mapToInt的pipeline后面可以是average,max,min,count,sum 
        double avg = users.parallelStream().mapToInt(User::getAge) 
                .average().getAsDouble(); 
  
        System.out.println("reduceAvg User Age: " + avg); 
    } 
  
    //获取所有用户的年龄总和 
    private static void reduceSum() { 
        double sum = users.parallelStream().mapToInt(User::getAge) 
                .reduce(0, (x, y) -> x + y); // 可以简写为.sum() 
  
        System.out.println("reduceSum User Age: " + sum); 
    } 
} 
```

# 参考
 [github/along101](https://github.com/along101/java-components-test/blob/master/java8/src/test/java/com/yzl/test/java8/stream/StreamTest.java)

