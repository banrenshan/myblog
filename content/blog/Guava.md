---
title: Guava
tags:
  - java
categories:
  - 技术
date: 2022-12-02 12:48:14
---

# 集合



## List操作

```java
        // 创建ArrayList
        ArrayList<String> arrayList1 = Lists.newArrayList();
        ArrayList<String> arrayList2 = Lists.newArrayList("1", "2", "3");
        ArrayList<String> arrayList3 = Lists.newArrayList(arrayList1);
        ArrayList<String> arrayList4 = Lists.newArrayListWithCapacity(10);

        //创建LinkedList
        LinkedList<Object> linkedList1 = Lists.newLinkedList();
        LinkedList<String> linkedList2 = Lists.newLinkedList(arrayList1);

        //创建CopyOnWriteArrayList
        CopyOnWriteArrayList<Object> copyOnWriteArrayList1 = Lists.newCopyOnWriteArrayList();
        CopyOnWriteArrayList<String> copyOnWriteArrayList2 = Lists.newCopyOnWriteArrayList(arrayList1);


        //反转list
        List<String> reverse = Lists.reverse(arrayList2);
        System.err.println(reverse); //[3, 2, 1]
        //切分list
        List<List<String>> partition = Lists.partition(arrayList2, 2);
        System.err.println(partition); //[[1, 2], [3]]

        //笛卡尔集
        List<List<String>> lists = Lists.cartesianProduct(Lists.newArrayList("A", "B"), arrayList2);
        System.err.println(lists);//[[A, 1], [A, 2], [A, 3], [B, 1], [B, 2], [B, 3]]

        //转化集合元素
        List<String> transform = Lists.transform(arrayList2, item -> "Prefix" + item);
        System.err.println(transform); //[Prefix1, Prefix2, Prefix3]

        //将字符串拆成单个字符
        ImmutableList<Character> abc = Lists.charactersOf("ABC");
        System.err.println(abc);//[A, B, C]
```

## Map操作

```java
        Map map = new HashMap();
        HashMap<Object, Object> hashMap1 = Maps.newHashMap();
        HashMap<Object, Object> hashMap2 = Maps.newHashMap(map);
        HashMap<Object, Object> hashMap3 = Maps.newHashMapWithExpectedSize(10);

        LinkedHashMap<Object, Object> linkedHashMap1 = Maps.newLinkedHashMap();
        LinkedHashMap<Object, Object> linkedHashMap2 = Maps.newLinkedHashMap(map);
        LinkedHashMap<Object, Object> linkedHashMap3 = Maps.newLinkedHashMapWithExpectedSize(10);

        ConcurrentMap<Object, Object> concurrentMap = Maps.newConcurrentMap();

        EnumMap<MAN, Object> manObjectEnumMap = Maps.newEnumMap(MAN.class);
        EnumMap enumMap = Maps.newEnumMap(map);

        TreeMap<Comparable, Object> treeMap1 = Maps.newTreeMap();
        TreeMap<Integer, String> treeMap2 = Maps.newTreeMap((a,b)->Integer.compare(a,b));

        IdentityHashMap<Object, Object> identityHashMap = Maps.newIdentityHashMap();
```

## Multimap 

一个key可以映射多个value的hashMap:

```java
        ArrayListMultimap<Object, Object> multimap = ArrayListMultimap.create();
        multimap.put("key",1);
        multimap.put("key",2);
        multimap.put("key",2);
        List<Object> value = multimap.get("key");
        System.err.println(value); //[1, 2, 2]
        Map<Object, Collection<Object>> objectCollectionMap = multimap.asMap();
        System.err.println(objectCollectionMap); //{key=[1, 2, 2]}
```

## BiMap 

一种连value也不能重复的HashMap

```java
        HashBiMap<Object, Object> biMap = HashBiMap.create();
        biMap.put("key","value");
       // biMap.put("key2","value"); 抛出异常
        biMap.forcePut("key2","value"); //{key2=value}
        System.err.println(biMap);

        BiMap<Object, Object> inverse = biMap.inverse();
        System.err.println(inverse); //{key2=value}
```

## Table

相当于mysql中的表：

```java
        HashBasedTable<Object, Object, Object> table = HashBasedTable.create();
        table.put("id_1","name","zzq");
        table.put("id_1","age","12");
        table.put("id_1","addr","henan");
        System.err.println(table); //{id_1={name=zzq, age=12, addr=henan}}

        Object o = table.get("id_1", "name");
        System.err.println(o); //zzq

        Map<Object, Object> addr = table.column("addr");
        System.err.println(addr); //{id_1=henan}
```

## Multiset 

一种用来计数的Set:

```java
        HashMultiset<Object> multiset = HashMultiset.create();
        multiset.add("apple");
        multiset.add("apple");
        multiset.add("orange");
        System.err.println(multiset.count("apple")); //2

        Set<Object> elementSet = multiset.elementSet();
        System.err.println(elementSet); // [orange, apple]
```
