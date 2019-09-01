---
layout: post
title: "ARTS 第5周 GSON源码分析"
date: 2019-04-22 00:58:05 +0800
comments: true
categories: ["ARTS", "Gson", "Duplicate Payment"]
---

## Algorithm

本周完成的算法题: [Trapping Rain Water](https://leetcode.com/problems/trapping-rain-water/)

题目要求:

```
给定一个n个大于等于0的整数，每个整数代表一个宽度为1的竖条的高度。计算其组成的图形可以在雨后保留多少单位的水
```

![rainwatertrap.png](https://assets.leetcode.com/uploads/2018/10/22/rainwatertrap.png)

<!-- more -->

这个是图代表数组`[0,1,0,2,1,0,1,3,2,1,2,1]`，在这个情况下，可以保留6个单位的雨水。

**分析:**

这个题我原始解决思路是尝试从左往右找到最大的一个蓄水区间。如下图(a)所示: 条件是 `right > left`，这样可以直接计算出**left**和**right**之间的蓄水量。当然还有一种情况如图(b)所示，`left`是剩下所有中最高的，那么我们需要找到右侧中最大的一个，把它当做**right**和**left**计算水数量。然后让`left = right`继续迭代。

![rainwatertrapv2.png](https://0bb6ac2.oss-cn-hongkong.aliyuncs.com/image/rainwatertrap.png?x-oss-process=style/black)

算法复杂的：在正常情况下是`O(N)`，最坏情况是`O(N^2)`，这个会在数组是倒序有序的情况下出现，因为每次都要往右循环到底才能确定**right**的位置。

```Java

    /**
    *  runtime 1ms, beats 99%
    */
    public int trap(int[] height) {

        if (height.length < 3) {
            return 0;
        }

        int total = 0;
        int left = 0;
        int maxLessThanLeftIndex = 0;
        int maxLessThanLeftValue = 0;
        int solid = 0;

        while (left < height.length - 2) {

            boolean cal = false;
            for (int i = left + 1; i < height.length; i++) {
                if (height[left] >= height[i]) {
                    solid += height[i];

                    //计算第二大的数，以便后续回溯使用
                    if (maxLessThanLeftValue <= height[i]) {
                        maxLessThanLeftIndex = i;
                        maxLessThanLeftValue = height[i];
                    }
                } else {
                    //在这里说明要计算蓄水了
                    total += (Math.min(height[left], height[i]) * (i - left - 1) - solid);
                    left = i;
                    cal = true;
                    break;
                }
            }

            if (!cal) {
                //到这里说明左边界是最大的值，需要使用第二大的数来做右边界。
                solid = 0;
                for (int i = left + 1; i < maxLessThanLeftIndex; i++) {
                    solid += height[i];
                }
                total += (Math.min(height[left], height[maxLessThanLeftIndex]) * (maxLessThanLeftIndex - left - 1) - solid);

                left = maxLessThanLeftIndex;
            }

            maxLessThanLeftIndex = 0;
            maxLessThanLeftValue = 0;
            solid = 0;
        }


        return total;
    }
```

在这个题的Solution中有一个更简洁的双指针版本，算法复杂度O(N)。其思想是单独为每个位置计算它的蓄水量，一个位置的蓄水量的大小是有它两侧最高点中较小的一个减去该位置的高度决定的(公式  = `min(left_max, right_max) - height`) ，比如我们在计算位置3的蓄水量时，它的left_max是位置1，right_max是位置10，根据公式位置3的蓄水量=`left_max - height`

![rainwatertrapv2](https://0bb6ac2.oss-cn-hongkong.aliyuncs.com/image/rainwatertrapv2.png?x-oss-process=style/black)

代码更简单:

```Java
    int trap2Points(int[] height) {
        int left = 0, right = height.length - 1;
        int ans = 0;
        int left_max = 0, right_max = 0;
        while (left < right) {
            if (height[left] < height[right]) {
                if (height[left] >= left_max) {
                    left_max = height[left];
                } else {
                    ans += (left_max - height[left]);
                }
                ++left;
            } else {
                if (height[right] >= right_max) {
                    right_max = height[right];
                } else {
                    ans += (right_max - height[right]);
                }
                --right;
            }
        }
        return ans;
    }

```

## Review

本周Review [Avoiding Double Payments in a Distributed Payments System](https://medium.com/airbnb-engineering/avoiding-double-payments-in-a-distributed-payments-system-2981f6b070bb)

本文介绍了Aribnb支付团队是如何分析和解决重复支付的问题。

### 背景介绍

Aribnb内部服务架构像SOA转型，这就导致原本在一个系统或者大服务里面完成的功能(比如下单和支付)，会被拆分成独立的服务。这样能够让开发人员更加专注且快速迭代，但是也引入了如何保证数据一致性的问题。

为了解决分布式一致性的问题，文中提到了三种解决方案：

* read repair. 在读取求的时候进行一致性的校验和补偿。[参考](https://docs.datastax.com/en/cassandra/3.0/cassandra/operations/opsRepairNodesReadRepair.html)。
    * 优点：客户端简单，无需关注一致性逻辑
    * 缺点：服务端需要额外的逻辑来保证一致性，会降低读的吞吐量。   
* write repair. 客户端通过重试的方式来保证数据一致。
    * 优点：服务端无需额外的逻辑保证一致性。
    * 缺点：客户端需要维护一套重试的逻辑。服务端需要保证重试请求的幂等性
* asynchronous repair. 通过后台程序，定时任务等检查数据一致性，并通知客户端结果。
    * 优点：与主逻辑解耦
    * 缺点：会有一定的延迟性

在实践上可以把`asynchronous repair`作为read repair和write repair的第二道防线。

### 幂等性[idempotency]

幂等性是下单&支付这类分布式应用中最基本的要求，必须保证每次请求`有且仅被处理一次`。如果不这样的话就会出现重复扣款的情况。同时幂等性允许一次操作多次重试，并且得到一致的结果。

![幂等性](https://0bb6ac2.oss-cn-hongkong.aliyuncs.com/image/1_bft4XyiErJha_uhbGLL6-w.png)

### 问题难点描述

* 需要设计一个通用支持多个支付服务的幂等性解决方案。
* 数据一致性不能妥协。这个会影响公司的形象
* 低延迟。需要分布式部署。
* 简单易用。业务开发不需要了解底层实现细节

### 解决方案

Airbnb支付团队实现了一个通用幂等库(`general-purpose idempotency library`)，取得名字叫[Orpheus](https://zh.wikipedia.org/wiki/%E4%BF%84%E8%80%B3%E7%94%AB%E6%96%AF)。(发现国内外取名都喜欢用希腊神话故事里面的任务)

这个库可以提供低延迟，并且与上下游良好隔离的实现。从整体来看，主要有以下几个概念：

* **idempotency key** 通过一个idempotency key来代表一次幂等的请求。
* **sharded master** 数据的读和写都走数据库Master节点
* **Database transactions** 使用`Java Lambdas`将分布在不同代码中的逻辑组合到一起保证原子性。

Orpheus将一次请求划分为三个阶段: Pre-RPC, RPS和Post-RPC:

* **Pre-RPC** 将请求详情持久化到数据库
* **RPC** 执行业务请求
* **Post-RPC** 将请求结果持久化的数据库。包括: 成功、失败以及是否可以重试等。

这里有两条基本原则：

1. 在Pre-RPC和Post-RPC阶段不会有业务逻辑
2. 在RPC阶段不会有数据库操作

通过这两个原则将网络通信和数据库操作隔离开来，避免处理因为网络带来的各种不确定的问题，比如超时等。同时为了保证一致性，框架将Pre-RPC和Post-RPC放到同一个事物中执行，这样可以保证这两个阶段一定是从事成功或者失败。以下是一次请求的示意图:

![请求流程图](https://0bb6ac2.oss-cn-hongkong.aliyuncs.com/image/1_EzRHQV1GaNO6fXg8NSQ2Cg.png)

### 异常处理

当出现异常时，框架需要明确告知客户端本次请求是可重试的还是不可重试的。Airbnb通过以下的策略将不同的异常划分为可重试或者不可重试.

![异常处理](https://0bb6ac2.oss-cn-hongkong.aliyuncs.com/image/1__q2kiqlR69N_Tybu37Px2Q.png)

### 客户端至关重要

在本文的场景下，客户端需要一些智能。他需要有以下几项关键职责：

1. 为每个请求生成一个唯一的`Idempotency Key`,并且在重试的时候复用原来的key
2. 请求之前持久化key
3. 合适的处理成功请求
4. 在不允许重试的时候要处理好请求压力
4. 设定合适的重试策略

### 怎么选择一个 Idempotency Key

有两种方式来生成一个Idempotency Key：

1. **request-level** 每次请求都使用一个随机且唯一的key。比如UUID，snowflake等
2. **entity-level** 包含了业务含义的key，比request-level的要更加严格。可以通过实体模型的关键字段来生成一个明确的key. 比如支付操作`1234`只能产生一次退款操作，那么可以定义类似`payment-1234-refund`的key来保证唯一性。

### 每个请求都有可过期的租约(Expiring Lease)

在请求处理之前需要先获取一个锁，拿到锁以后才可以继续处理数据。锁一般要有超时时间，当超时时间到了以后可以发起重试。

推荐锁的超时间要比RPC请求的超时时间长。否则可能会有重复请求。

### 坚持使用数据库Master节点

Orpheus通过数据来保存`Idempotency Key`等重要数据，为了保证数据一致性所有读写操作都在Master节点上执行。避免主从结构中因为数据复制可能出现的重复操作。以下是主从结构中可能出现的重复操作示意图:

![主从有问题](https://0bb6ac2.oss-cn-hongkong.aliyuncs.com/image/1_ASFZE0BWYFYxokg1WUgnZw.png)

只用主节点就不会出现因为数据复制导致可能的重复操作: 

![只用主没问题](https://0bb6ac2.oss-cn-hongkong.aliyuncs.com/image/1_FBLUYnSjxQ15uSdAisvriA.png)

如果只用一个主节点的话，主节点就会成为整个系统的瓶颈。所以系统设计通过key分片的方式部署多个主节点，避免一个主节点过热的问题。

### 总结

想要做到数据一致性就会引入额外的复杂度。没有付出就没有回报。
Orpheus目前达到了5个9的一致性。

注意: Review中出现的图片均引用自[Avoiding Double Payments in a Distributed Payments System](https://medium.com/airbnb-engineering/avoiding-double-payments-in-a-distributed-payments-system-2981f6b070bb)，版权归它所有。

## Tip

本周分享一个线上排查问题时查看占用CPU高的前几个线程。

```
ps -eL -o pid,%cpu,lwp | grep -i <pid> | sort -nr -k2 | awk '{printf("%s %s %x\n", $1,$2,$3)}' | head
```

## Share

**Gson源码阅读(一)**

`Gson`是开发中常用的`Json`库，使用起来非常简单。整个[工程的源码](https://github.com/google/gson)不多，非常适合作为源码阅读的开胃菜。

`Gson`的核心类不多，整体类结构如下图(去掉了一些具体的实现类和异常类):

![gson类图](https://0bb6ac2.oss-cn-hongkong.aliyuncs.com/image/gson%E7%B1%BB%E5%9B%BE.jpg?x-oss-process=style/black)

主要分以下几块：

* **JsonElement(以及子类)** 代表了Json的中的几种类型：`对象`，`数组`，`基本类型`(在Java中主包括: String, Java基本类型以及基本类型的包装类)，`Null`
* **JsonReader和JsonWriter** 封装了Gson读取和输出Json数据的过程。
* **TypeAdapter和TypeAdapterFactory** Gson的核心部分，通过TypeAdapter将具体类型的转换细节封装起来，使Gson有良好的灵活性和扩展性。
* **JsonSerializer和JsonDeserializer** 这里两个接口允许你自定义某个类型的序列化逻辑。不过现在推荐使用**TypeAdapter**的方式


可以看到Gson的核心逻辑是很简单的`^_^`。 

第一篇首先通过简单的描述下 `new Gson().toJson(new A())` 流程来说明下Gson的核心实现，后续再继续深入其他的细节。

![流程](https://0bb6ac2.oss-cn-hongkong.aliyuncs.com/image/ARTS-gson-%E6%B5%81%E7%A8%8B.png?x-oss-process=style/black)

### 创建Gson对象

最简单的用法中创建`Gson`对象: `Gson gson = new Gson()`，但是实际上内部做的事情远比看上去的多：

```Java

public Gson() {
    this(Excluder.DEFAULT, FieldNamingPolicy.IDENTITY,
        Collections.<Type, InstanceCreator<?>>emptyMap(), DEFAULT_SERIALIZE_NULLS,
        DEFAULT_COMPLEX_MAP_KEYS, DEFAULT_JSON_NON_EXECUTABLE, DEFAULT_ESCAPE_HTML,
        DEFAULT_PRETTY_PRINT, DEFAULT_LENIENT, DEFAULT_SPECIALIZE_FLOAT_VALUES,
        LongSerializationPolicy.DEFAULT, null, DateFormat.DEFAULT, DateFormat.DEFAULT,
        Collections.<TypeAdapterFactory>emptyList(), Collections.<TypeAdapterFactory>emptyList(),
        Collections.<TypeAdapterFactory>emptyList());
  }

//实际的
Gson(Excluder excluder, FieldNamingStrategy fieldNamingStrategy,
      Map<Type, InstanceCreator<?>> instanceCreators, boolean serializeNulls,
      boolean complexMapKeySerialization, boolean generateNonExecutableGson, boolean htmlSafe,
      boolean prettyPrinting, boolean lenient, boolean serializeSpecialFloatingPointValues,
      LongSerializationPolicy longSerializationPolicy, String datePattern, int dateStyle,
      int timeStyle, List<TypeAdapterFactory> builderFactories,
      List<TypeAdapterFactory> builderHierarchyFactories,
      List<TypeAdapterFactory> factoriesToBeAdded) {
    // 执行排查的字段和类型  
    this.excluder = excluder;
    // 字段命名机制。比如是 aName 后者是 a_name
    this.fieldNamingStrategy = fieldNamingStrategy;
    // Gson默认是需要有不带参数的构造函数，但是碰到没有无参数构造函数且没有权限修改第三方类的情况下，可以通过InstanceCreator来解决
    this.instanceCreators = instanceCreators;
    // 通过构造器创建指定的类型
    this.constructorConstructor = new ConstructorConstructor(instanceCreators);
    // 是否序列化null
    this.serializeNulls = serializeNulls;
    this.complexMapKeySerialization = complexMapKeySerialization;
    // 是否输出不可执行的json。如果为true，会在开头先输出一个")]}'\n"
    this.generateNonExecutableJson = generateNonExecutableGson;
    // 是否输出Html安全的JSON. 如果为true，在输出前会转义 < > & =
    this.htmlSafe = htmlSafe;
    this.prettyPrinting = prettyPrinting;
    // 是否使用宽松的模式。如果为true，则会忽略某些限制
    this.lenient = lenient;
    this.serializeSpecialFloatingPointValues = serializeSpecialFloatingPointValues;
    // 对long和Long类型的序列化策略
    this.longSerializationPolicy = longSerializationPolicy;
    // 日期的模式
    this.datePattern = datePattern;
    this.dateStyle = dateStyle;
    this.timeStyle = timeStyle;
    // 创建TypeAdapter的工厂类
    this.builderFactories = builderFactories;
    this.builderHierarchyFactories = builderHierarchyFactories;

    List<TypeAdapterFactory> factories = new ArrayList<TypeAdapterFactory>();

    // built-in type adapters that cannot be overridden
    factories.add(TypeAdapters.JSON_ELEMENT_FACTORY);
    factories.add(ObjectTypeAdapter.FACTORY);

    // the excluder must precede all adapters that handle user-defined types
    factories.add(excluder);

    // users' type adapters
    factories.addAll(factoriesToBeAdded);

    // type adapters for basic platform types
    factories.add(TypeAdapters.STRING_FACTORY);
    factories.add(TypeAdapters.INTEGER_FACTORY);
    factories.add(TypeAdapters.BOOLEAN_FACTORY);
    factories.add(TypeAdapters.BYTE_FACTORY);
    factories.add(TypeAdapters.SHORT_FACTORY);
    TypeAdapter<Number> longAdapter = longAdapter(longSerializationPolicy);
    factories.add(TypeAdapters.newFactory(long.class, Long.class, longAdapter));
    factories.add(TypeAdapters.newFactory(double.class, Double.class,
            doubleAdapter(serializeSpecialFloatingPointValues)));
    factories.add(TypeAdapters.newFactory(float.class, Float.class,
            floatAdapter(serializeSpecialFloatingPointValues)));
    factories.add(TypeAdapters.NUMBER_FACTORY);
    factories.add(TypeAdapters.ATOMIC_INTEGER_FACTORY);
    factories.add(TypeAdapters.ATOMIC_BOOLEAN_FACTORY);
    factories.add(TypeAdapters.newFactory(AtomicLong.class, atomicLongAdapter(longAdapter)));
    factories.add(TypeAdapters.newFactory(AtomicLongArray.class, atomicLongArrayAdapter(longAdapter)));
    factories.add(TypeAdapters.ATOMIC_INTEGER_ARRAY_FACTORY);
    factories.add(TypeAdapters.CHARACTER_FACTORY);
    factories.add(TypeAdapters.STRING_BUILDER_FACTORY);
    factories.add(TypeAdapters.STRING_BUFFER_FACTORY);
    factories.add(TypeAdapters.newFactory(BigDecimal.class, TypeAdapters.BIG_DECIMAL));
    factories.add(TypeAdapters.newFactory(BigInteger.class, TypeAdapters.BIG_INTEGER));
    factories.add(TypeAdapters.URL_FACTORY);
    factories.add(TypeAdapters.URI_FACTORY);
    factories.add(TypeAdapters.UUID_FACTORY);
    factories.add(TypeAdapters.CURRENCY_FACTORY);
    factories.add(TypeAdapters.LOCALE_FACTORY);
    factories.add(TypeAdapters.INET_ADDRESS_FACTORY);
    factories.add(TypeAdapters.BIT_SET_FACTORY);
    factories.add(DateTypeAdapter.FACTORY);
    factories.add(TypeAdapters.CALENDAR_FACTORY);
    factories.add(TimeTypeAdapter.FACTORY);
    factories.add(SqlDateTypeAdapter.FACTORY);
    factories.add(TypeAdapters.TIMESTAMP_FACTORY);
    factories.add(ArrayTypeAdapter.FACTORY);
    factories.add(TypeAdapters.CLASS_FACTORY);

    // type adapters for composite and user-defined types
    factories.add(new CollectionTypeAdapterFactory(constructorConstructor));
    factories.add(new MapTypeAdapterFactory(constructorConstructor, complexMapKeySerialization));
    this.jsonAdapterFactory = new JsonAdapterAnnotationTypeAdapterFactory(constructorConstructor);
    factories.add(jsonAdapterFactory);
    factories.add(TypeAdapters.ENUM_FACTORY);
    
    //在通常情况下，我们自定义的类型会通过这个 ReflectiveTypeAdapter 来做转换。
    factories.add(new ReflectiveTypeAdapterFactory(
        constructorConstructor, fieldNamingStrategy, excluder, jsonAdapterFactory));

    this.factories = Collections.unmodifiableList(factories);
  }
```

可以看到`Gson`提供了很多的开关和扩展点来定制行为。

### 创建Reader/Writer

创建JsonReader 和 JsonWriter。这个会根据Json标准来读取和输出json数据，具体的细节本文先不深入研究，下篇文章深入研究下。

```Java
  public JsonWriter newJsonWriter(Writer writer) throws IOException {
    if (generateNonExecutableJson) {
      writer.write(JSON_NON_EXECUTABLE_PREFIX);
    }
    JsonWriter jsonWriter = new JsonWriter(writer);
    if (prettyPrinting) {
      jsonWriter.setIndent("  ");
    }
    jsonWriter.setSerializeNulls(serializeNulls);
    return jsonWriter;
  }

  public JsonReader newJsonReader(Reader reader) {
    JsonReader jsonReader = new JsonReader(reader);
    jsonReader.setLenient(lenient);
    return jsonReader;
  }
```

### 获取类型对应的TypeAdapter

```Java
  public <T> TypeAdapter<T> getAdapter(TypeToken<T> type) {
    // 首先从本地内存获取
    TypeAdapter<?> cached = typeTokenCache.get(type == null ? NULL_KEY_SURROGATE : type);
    if (cached != null) {
      return (TypeAdapter<T>) cached;
    }

    // 接着从ThreadLocal中获取
    Map<TypeToken<?>, FutureTypeAdapter<?>> threadCalls = calls.get();
    boolean requiresThreadLocalCleanup = false;
    if (threadCalls == null) {
      threadCalls = new HashMap<TypeToken<?>, FutureTypeAdapter<?>>();
      calls.set(threadCalls);
      requiresThreadLocalCleanup = true;
    }

    // the key and value type parameters always agree
    FutureTypeAdapter<T> ongoingCall = (FutureTypeAdapter<T>) threadCalls.get(type);
    if (ongoingCall != null) {
      return ongoingCall;
    }

    try { 
      FutureTypeAdapter<T> call = new FutureTypeAdapter<T>();
      threadCalls.put(type, call);

      // 循环左右的 TypeAdapterFactory 找到可以处理当前类型的第一个 Factory
      for (TypeAdapterFactory factory : factories) {
        TypeAdapter<T> candidate = factory.create(this, type);
        if (candidate != null) {
          call.setDelegate(candidate);
          typeTokenCache.put(type, candidate);
          return candidate;
        }
      }
      throw new IllegalArgumentException("GSON (" + GsonBuildConfig.VERSION + ") cannot handle " + type);
    } finally {
      threadCalls.remove(type);

      if (requiresThreadLocalCleanup) {
        calls.remove();
      }
    }
  }
```

前面提到在没有特定指定的情况下，自定义的类型都是由`ReflectiveTypeAdapterFactory`来处理的，让我们来看下`ReflectiveTypeAdapterFactory`的`create`方法。

```Java

public <T> TypeAdapter<T> create(Gson gson, final TypeToken<T> type) {
    Class<? super T> raw = type.getRawType();

    if (!Object.class.isAssignableFrom(raw)) {
      // 到这里的话说明是基本类型
      return null; 
    }

    // 为指定类型创建一个构造器工厂。 这里会优先使用前面提到的 InstanceCreator.
    ObjectConstructor<T> constructor = constructorConstructor.get(type);
    
    // 为类型的每个字段映射成内部的 BoundField对象。
    return new Adapter<T>(constructor, getBoundFields(gson, type, raw));
  }
```

`BoundField` 是一个重要的对象，定义了一个字段该如何读取和输出。来看下它的定义：

```Java

  static abstract class BoundField {
    final String name;
    final boolean serialized;
    final boolean deserialized;

    protected BoundField(String name, boolean serialized, boolean deserialized) {
      this.name = name;
      this.serialized = serialized;
      this.deserialized = deserialized;
    }
    
    // 判断一个字段是否可写
    abstract boolean writeField(Object value) throws IOException, IllegalAccessException;
    // 将字段写入到writer
    abstract void write(JsonWriter writer, Object value) throws IOException, IllegalAccessException;
    // 从输入中读取字段的值
    abstract void read(JsonReader reader, Object value) throws IOException, IllegalAccessException;
  }
```

来看看`BoundField`是如何构建出来的：

```Java
private Map<String, BoundField> getBoundFields(Gson context, TypeToken<?> type, Class<?> raw) {
    Map<String, BoundField> result = new LinkedHashMap<String, BoundField>();
    
    // 如果是接口直接返回
    if (raw.isInterface()) {
      return result;
    }

    Type declaredType = type.getType();
    while (raw != Object.class) {
      // 获取当前类型声明的字段
      Field[] fields = raw.getDeclaredFields();
      for (Field field : fields) {
        
        //判断字段是否需要序列化
        boolean serialize = excludeField(field, true);
        boolean deserialize = excludeField(field, false);
        if (!serialize && !deserialize) {
          continue;
        }
        accessor.makeAccessible(field);
        Type fieldType = $Gson$Types.resolve(type.getType(), raw, field.getGenericType());
        List<String> fieldNames = getFieldNames(field);
        BoundField previous = null;
        for (int i = 0, size = fieldNames.size(); i < size; ++i) {
          String name = fieldNames.get(i);
          if (i != 0) serialize = false; // only serialize the default name
          BoundField boundField = createBoundField(context, field, name,
              TypeToken.get(fieldType), serialize, deserialize);
          BoundField replaced = result.put(name, boundField);
          if (previous == null) previous = replaced;
        }
        if (previous != null) {
          throw new IllegalArgumentException(declaredType
              + " declares multiple JSON fields named " + previous.name);
        }
      }
      
      //继续处理父类
      type = TypeToken.get($Gson$Types.resolve(type.getType(), raw, raw.getGenericSuperclass()));
      raw = type.getRawType();
    }
    return result;
  }
  
private ReflectiveTypeAdapterFactory.BoundField createBoundField(
      final Gson context, final Field field, final String name,
      final TypeToken<?> fieldType, boolean serialize, boolean deserialize) {
        
    final boolean isPrimitive = Primitives.isPrimitive(fieldType.getRawType());
    
    // 为字段关联TypeAdapter. 首先看是否有通过注解特别指定的，没有的话通过getAdapter来创建
    JsonAdapter annotation = field.getAnnotation(JsonAdapter.class);
    TypeAdapter<?> mapped = null;
    if (annotation != null) {
      mapped = jsonAdapterFactory.getTypeAdapter(
          constructorConstructor, context, fieldType, annotation);
    }
    final boolean jsonAdapterPresent = mapped != null;
    if (mapped == null) mapped = context.getAdapter(fieldType);

    final TypeAdapter<?> typeAdapter = mapped;
    return new ReflectiveTypeAdapterFactory.BoundField(name, serialize, deserialize) {
      
      @Override void write(JsonWriter writer, Object value)
          throws IOException, IllegalAccessException {
        Object fieldValue = field.get(value);
        TypeAdapter t = jsonAdapterPresent ? typeAdapter
            : new TypeAdapterRuntimeTypeWrapper(context, typeAdapter, fieldType.getType());
        t.write(writer, fieldValue);
      }
      @Override void read(JsonReader reader, Object value)
          throws IOException, IllegalAccessException {
        Object fieldValue = typeAdapter.read(reader);
        if (fieldValue != null || !isPrimitive) {
          field.set(value, fieldValue);
        }
      }
      @Override public boolean writeField(Object value) throws IOException, IllegalAccessException {
        if (!serialized) return false;
        Object fieldValue = field.get(value);
        return fieldValue != value; 
      }
    };
  }
```

至此，已经解析好了该类以及父类的所有字段。接下来看下是TypeAdapter是怎么执行转换的。

### TypaAdapter执行转换

这块从`ReflectiveTypeAdapterFactory`的实现来看就比较简单了，通过对每个解析好的`BoundField` 调用它的`read`和`write`方法即可。

```Java
public void toJson(Object src, Type typeOfSrc, JsonWriter writer) throws JsonIOException {
    TypeAdapter<?> adapter = getAdapter(TypeToken.get(typeOfSrc));
    boolean oldLenient = writer.isLenient();
    writer.setLenient(true);
    boolean oldHtmlSafe = writer.isHtmlSafe();
    writer.setHtmlSafe(htmlSafe);
    boolean oldSerializeNulls = writer.getSerializeNulls();
    writer.setSerializeNulls(serializeNulls);
    try {
      ((TypeAdapter<Object>) adapter).write(writer, src);
    } catch (IOException e) {
      throw new JsonIOException(e);
    } catch (AssertionError e) {
      throw error;
    } finally {
      writer.setLenient(oldLenient);
      writer.setHtmlSafe(oldHtmlSafe);
      writer.setSerializeNulls(oldSerializeNulls);
    }
  }
  
public <T> T fromJson(JsonReader reader, Type typeOfT) throws JsonIOException, JsonSyntaxException {
    boolean isEmpty = true;
    boolean oldLenient = reader.isLenient();
    reader.setLenient(true);
    try {
      reader.peek();
      isEmpty = false;
      TypeToken<T> typeToken = (TypeToken<T>) TypeToken.get(typeOfT);
      TypeAdapter<T> typeAdapter = getAdapter(typeToken);
      T object = typeAdapter.read(reader);
      return object;
    } catch (EOFException e) {
      throw new JsonSyntaxException(e);
    } catch (IllegalStateException e) {
      throw new JsonSyntaxException(e);
    } catch (IOException e) {
      throw new JsonSyntaxException(e);
    } catch (AssertionError e) {
      throw error;
    } finally {
      reader.setLenient(oldLenient);
    }
  }
```

### 总结

从代码中可以看到，Gson针对所有的类型细节的处理都抽象封装到了TypeAdapter中，这样使得主流程可以不用关心转换细节，同时也为用户扩展提供了无限的可能。



