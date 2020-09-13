---
title: 打怪之流式编程：Stream（一）
date: 2020-09-08 10:58:52
tags: 
- Stream
categories: "java基础"  
---
## java中的流式编程——Stream API

```txt
Stream不是集合元素，它不是数据结构并不保存数据，它更像一个高级版本的 Iterator,单向且不可往复，数据只能遍历一次，遍历过一次后即用尽了，就好比流水从面前流过，一去不复返。
```

### 流的构成

当我们使用一个流的时候，通常包括三个基本步骤：

1. 获取一个数据源（source）
2. 数据转换: 执行操作获取想要的结果，每次转换原有 Stream 对象不改变，返回一个新的 Stream 对象（可以有多次转换），这就允许对其操作可以像链一样排列，变成一个管道。
3. 终止操作

### 1、操作的数据源
```java
   public static List<Employee> generateListData() {
        return Arrays.asList(new Employee("Matt", 5000, "New York"),
                new Employee("Steve", 6000, "London"),
                new Employee("Steve", 7800, "Shanghai"),
                new Employee("Carrie", 10000, "New York"),
                new Employee("Peter", 7000, "New York"),
                new Employee("Alec", 6000, "London"),
                new Employee("Sarah", 8000, "London"),
                new Employee("Rebecca", 4000, "New York"),
                new Employee("Pat", 20000, "New York"),
                new Employee("Tammy", 9000, "New York"),
                new Employee("Fred", 15000, "Tokyo"));
    }
```
### 2.1、过滤（筛选与切片）

  方法 | 示意
----|------
filter | 接收Lambda表达式，从流中排除某些元素
distinct| 返回包含唯一元素的流（唯一性取决于元素相等的实现方式）,例如通过流所生成元素的 hashCode() 和 equals() 去除重复元素
limit| 截断流，使其元素不超过给定数量
skip| 返回一个丢弃前n个元素的流

![avatar](1.png)<br>

```java
 /**
     　* @Description: 过滤工资大于8000的所有员工
     　* @param
     　* @return
     */
    @Test
    public void test1(){
        List<Employee> results = generateListData();
        //old
        List<Employee> list = new ArrayList<>();
        for (Employee employee : results){
            if(employee.getSalary() > 8000){
                list.add(employee);
            }
        }
        System.out.println(list);

        //new 1
        List<Employee> collect = results.stream().filter(employee -> (employee.getSalary() > 8000))
                .collect(Collectors.toList());
        System.out.println(collect);

        //new 2
        List<Employee> collect1 = results.stream().filter(employee -> {
            if (employee.getSalary() > 8000) {
                return true;
            } else {
                return false;
            }
        }).collect(Collectors.toList());
        System.out.println(collect1);
    }
```
```
[
    Employee(name=Carrie pit, salary=10000, office=New York), 
    Employee(name=Pat, salary=20000, office=New York),
    Employee(name=Tammy, salary=9000, office=New York), 
    Employee(name=Fred, salary=15000, office=Tokyo)]
]
```

### 2.2、映射

  方法 | 示意 | 应用
----|------|--------
map | 应用于单个元素，将其映射成新元素（传递一个函数对象作为方法，把流中的元素转换成另一种类型）| map生成的是个一对一映射，比较常用
flatMap | 接收一个函数作为参数，将流中的每个值都换成另一个流，然后把所有流连接成一个流 | flatMap生成一个一对多映射

![avatar](2.png)<br>

```java
@Test
    public void test2() {
        List<Employee> results = generateListData();
        //old
        List<String> list = new ArrayList<>();
        for (Employee employee : results){
            if(employee.getSalary() > 8000){
                list.add(employee.getName());
            }
        }
        System.out.println(list);

        //new 1
        List<String> collect = results.stream()
                //过滤
                .filter(employee -> (employee.getSalary() > 8000))
                //转化映射
                .map(employee -> employee.getName()).collect(Collectors.toList());
        System.out.println(collect);

        //new 2
        List<String> collect1 = results.stream()
                //过滤
                .filter(employee -> (employee.getSalary() > 8000))
                //转化映射
                .map(Employee::getName).collect(Collectors.toList());
        System.out.println(collect1);
    }
```
```
[Carrie pit, Pat, Tammy, Fred]
```

![avatar](3.png)<br>

```java
  /**
    　* @Description: map 和 flatMap
    　* @param
    　* @return
  */
 @Test
    public void test3() {
        List<Employee> results = generateListData();
        //返回类型不一样
        List<String> collect = results.stream().
                //每一个对象映射为名字(名字数组进一步处理为流对象)
                flatMap(employee -> Arrays.stream(employee.getName().split(" ")))
                .collect(Collectors.toList());
        System.out.println(collect);

        List<Stream<String>> collect1 = results.stream().
                map(employee -> Arrays.stream(employee.getName().split(" ")))
                .collect(Collectors.toList());
        System.out.println(collect1);

        //--------------------------------------------------------

        //实现1
        List<String> collect2 = results.stream()
                .map(employee -> (employee.getName().split(" ")))
                .flatMap(Arrays::stream)
                .collect(Collectors.toList());
        System.out.println(collect2);

        //实现1
        List<String> collect3 = results.stream()
                //返回多个list
                .map(employee -> (employee.getName().split(" ")))
                //将流中的每个值都换成另一个流，然后把所有流连接成一个流
                .flatMap(str -> Arrays.asList(str).stream())
                .collect(Collectors.toList());
        System.out.println(collect3);
    }
```

```txt
map和flatMap的区别：flatMap的可以处理更深层次的数据，入参为多个list，结果可以返回为一个list，而map是一对一的，入参是多个list，结果返回必须是多个list。通俗的说，如果入参都是对象，那么flatMap可以操作对象里面的对象，而map只能操作第一层。
```

### 3、Collectors（返回流操作完之后的结果）
```txt
collect方法是一个结束操作，它可以使流里面的所有元素聚集到汇总结果。传递给collect方法参数是一个java.util.stream.Collector类型的对象。Collector对象实际上定义了一个如何把流中的元素聚集到最终结果的方法。(?).collect(Collectors.?)
```
  方法 | 示意
----|------
toList | 转化为List
toSet| 转化为Set
toMap|转化为Map

```java
 @Test
    public void test4(){

        List<Employee> employees = generateListData();

        //toList
        List<String> collect = employees.stream()
                .map(Employee::getName)
                .collect(Collectors.toList());
        System.out.println(collect);

        //toSet
        Set<String> collect1 = employees.stream()
                .map(Employee::getName)
                .collect(Collectors.toSet());
        System.out.println(collect1);

        //toMap
        Map<String, Employee> map1 = employees.stream()
                .collect(Collectors.toMap((key->key.getName()), (Person -> Person)));
        Map<String, Integer> map2 = employees.stream()
                .collect(Collectors.toMap((key->key.getName()), (Person -> Person.getSalary())));

        map1.forEach((key,value) -> System.out.println("key:" + key + "   value:"+ value));
        map2.forEach((key,value) -> System.out.println("key:" + key + "   value:"+ value));
    }
```

### Optional

```txt
Optional类主要解决的问题是臭名昭著的空指针异常（NullPointerException）—— 每个Java程序员都非常了解的异常。
本质上，这是一个包含有可选值的包装类，这意味着 Optional 类既可以含有对象也可以为空。
```

 方法 | 示意
----|------
Optional.of(T) | T为非空，否则初始化报错。你应该明确对象T不为null的时候使用 of()
Optional.ofNullable(T)| T为任意，可以为空。如果对象T即可能是null也可能是非null,你就应该使用ofNullable()方法：
isPresent()| 等价于 !=null
ifPresent(T)|  T可以是一段lambda表达式 ，或者其他代码，非空则执行

```java
@Test
    public void test5(){
        Employee employee = new Employee();
        employee.setName("oyj");
        //of(T),T为非空值
        Optional<Employee> emp = Optional.of(employee);
        System.out.println(emp.isPresent()?emp.get():"emp对象为空");

        //ofNullable(T),T为任意值
        Optional<String> name = Optional.ofNullable(employee.getName());
        //name!=null输出name;否则输出"name值是空的"
        System.out.println(name.isPresent()?name.get():"name值是空的");

        //如果不为空则打印,ifPresent(T),T可以是一段lambda表达式,或者其他代码，非空则执行
        Optional.ofNullable("oyj").ifPresent(x->{
            System.out.println(x+" isPresent");
        });

        //在optional为空值的情况下orElse和orElseGet都会执行，
        //当optional不为空时，orElse()方法仍然执行,orElseGet()不会执行。
        //在执行较密集的调用时，比如调用Web服务或数据查询，这个差异会对性能产生重大影响。

        //如果为null，则返回指定的字符串
        System.out.println(Optional.ofNullable(null).orElse("为空"));
        System.out.println(Optional.ofNullable("oyj").orElse("为空"));

        //如果为null，则返回指定的方法
        System.out.println(Optional.ofNullable(null).orElseGet(()->{
            return "hahahah";
        }));
        System.out.println(Optional.ofNullable(1).orElseGet(()->{
            return 2;
        }));
    }
```


