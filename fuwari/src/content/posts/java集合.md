---
title: java集合
published: 2026-09-05
description: 'bg：最近刷刷法老是遇到不会写的集合，于是在这里借助ai进行弱点集中突破'
image: ''
tags: [java]
category: '笔记'
draft: false 
lang: ''
---

# Java 常用集合与 API 详解

> 面向 Java 后端面试与日常开发的集合框架系统性整理文档。内容涵盖 `Collection` 与 `Map` 两大体系下的核心实现类、底层数据结构、关键 API、源码级原理、性能对比与并发场景实践，并补充 Java 8 引入的 `Stream` API 与 `java.util.concurrent` 包下的并发集合。**对应 JDK 版本以 JDK 8 / 11 / 17 为主**，关键差异会单独标注。

---

## 一、Java 集合框架总览

### 1.1 整体架构

Java 集合框架（Java Collections Framework, JCF）位于 `java.util` 包下，提供了一套统一的接口与实现，让数据的存储、检索、操作可复用、可互操作。整体结构可分为**两大根接口**：

- `java.util.Collection`：存储**单列数据**，每个元素独立存在
  - `List`：有序、可重复
  - `Set`：无序（部分实现有序）、不可重复
  - `Queue / Deque`：队列与双端队列
- `java.util.Map`：存储**双列数据**（key-value 键值对）

```text
                         Iterable<E>
                             │
                       Collection<E>
              ┌──────────────┼────────────────────┐
            List<E>       Set<E>            Queue<E>
              │             │                  │
       ┌──────┼──────┐   ┌──┼──┐          ┌───┼────┐
   ArrayList  LinkedList HashSet TreeSet  PriorityQueue Deque
       Vector  Stack     LinkedHashSet  ArrayDeque
                                  │
                                  └──> NavigableSet -> SortedSet
```

```text
                                Map<K,V>
                                  │
                ┌─────────────────┼─────────────────┐
            HashMap           SortedMap          Hashtable
                │                │                  │
        LinkedHashMap        TreeMap          Properties
                │
        WeakHashMap / IdentityHashMap
                │
        ConcurrentMap
                │
        ConcurrentHashMap
```

### 1.2 接口规范与设计思想

| 设计原则 | 体现 |
| --- | --- |
| **面向接口编程** | 业务代码应面向 `List` / `Map` 等接口编程，而不是具体实现 |
| **可选操作（Optional Operation）** | 接口中部分方法如 `add(int, E)` 标注 `UnsupportedOperationException`，未实现时可抛出（不可变集合常用） |
| **equals / hashCode 契约** | `Set` 与 `Map` 的 key 要求正确实现这俩方法，否则去重与查找失效 |
| **fail-fast 机制** | 多数集合在迭代过程中检测到结构性修改会抛 `ConcurrentModificationException`（`ConcurrentHashMap` 等是 fail-safe） |

### 1.3 包路径速查

| 包 | 主要内容 |
| --- | --- |
| `java.util` | 集合框架主体（`List` / `Set` / `Map` / `Queue` / 工具类） |
| `java.util.concurrent` | 并发集合（`ConcurrentHashMap` / `CopyOnWriteArrayList` / `BlockingQueue` 等） |
| `java.util.concurrent.atomic` | 原子类（不属集合但常配合并发集合使用） |
| `java.util.function` | 函数式接口（`Predicate` / `Function` / `Supplier` 等，配合 Stream 使用） |
| `java.util.stream` | Stream API（`Stream` / `Collectors`） |
| `java.util.Collections` | 集合工具类（旧 API） |

### 1.4 集合与数组的区别

| 维度 | 数组 | 集合 |
| --- | --- | --- |
| 长度 | 固定 | 动态扩容 |
| 类型 | 可存基本类型与对象 | 只能存对象（基本类型需装箱） |
| 功能 | 仅有 length 属性 | 提供丰富的 CRUD、批量操作、流式 API |
| 性能 | 极致紧凑（内存连续） | 一般略低于数组，但接口与实现优化已接近数组 |

### 1.5 通用遍历方式

```java
List<String> list = Arrays.asList("a", "b", "c");

// 1. for 循环（List 专属）
for (int i = 0; i < list.size(); i++) {
    System.out.println(list.get(i));
}

// 2. 增强 for（foreach，本质是 Iterator）
for (String s : list) {
    System.out.println(s);
}

// 3. Iterator（可在遍历中安全 remove）
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    if ("b".equals(s)) it.remove();
}

// 4. forEach + Lambda（Java 8+）
list.forEach(System.out::println);

// 5. Stream（Java 8+，可链式 + 并行）
list.stream().filter(s -> s.length() > 0).forEach(System.out::println);
```

---

## 二、List 接口详解

`List` 接口继承自 `Collection`，特征是**有序（按插入顺序）**、**可重复**、**可按索引访问**。

### 2.1 List 接口核心 API

```java
// 位置访问
E get(int index);
E set(int index, E element);    // 返回旧值
void add(int index, E element);
E remove(int index);

// 查找（基于 equals）
int indexOf(Object o);          // 第一次出现的下标，无则返回 -1
int lastIndexOf(Object o);

// 范围视图（重要！返回的是原集合的视图，修改互相影响）
List<E> subList(int fromIndex, int toIndex);

// 批量操作
boolean addAll(int index, Collection<? extends E> c);

// Java 8+ 默认方法
default void replaceAll(UnaryOperator<E> operator);   // 原地替换
default void sort(Comparator<? super E> c);           // 原地排序
```

### 2.2 ArrayList

#### 2.2.1 底层结构与关键字段

`ArrayList` 是基于**动态数组**实现的，**随机访问快**（O(1)）、**插入删除慢**（平均 O(n)，涉及数组拷贝）、**非线程安全**。

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable {

    private static final int DEFAULT_CAPACITY = 10;       // 初始容量
    private static final Object[] EMPTY_ELEMENTDATA = {};  // 空数组常量
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {}; // 区分无参构造
    transient Object[] elementData;  // 存储元素的数组，未被序列化
    private int size;                 // 实际元素数量
    protected transient int modCount = 0;  // 结构修改计数（fail-fast）
}
```

#### 2.2.2 三种构造方式

```java
new ArrayList<>();                 // elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA
new ArrayList<>(int initialCapacity);
new ArrayList<>(Collection<? extends E> c);
```

无参构造**不会**立即分配 10 个容量的数组，而是把内部数组设为一个空数组常量。**首次 `add` 时才扩容到 10**——这是一种懒加载策略。

#### 2.2.3 扩容机制（核心）

```java
public boolean add(E e) {
    modCount++;
    add(e, elementData, size);
}

private void add(E e, Object[] elementData, int s) {
    if (s == elementData.length)  // 容量已满，需扩容
        elementData = grow();
    elementData[s] = e;
    size = s + 1;
}

private Object[] grow() {
    return grow(size + 1);
}

private Object[] grow(int minCapacity) {
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);  // 1.5 倍
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)               // 接近 Integer.MAX_VALUE
        newCapacity = hugeCapacity(minCapacity);
    return elementData = Arrays.copyOf(elementData, newCapacity);
}
```

要点：
1. 每次扩容为原容量的 **1.5 倍**（`oldCapacity + oldCapacity >> 1`）。
2. 通过 `Arrays.copyOf` 复制数组，开销是 O(n)，所以**预判容量一次性分配**可以显著减少扩容次数。
3. `hugeCapacity` 在 `minCapacity < 0`（即溢出）时抛 `OutOfMemoryError`。

#### 2.2.4 常用 API 与复杂度

| 方法 | 时间复杂度 | 说明 |
| --- | --- | --- |
| `get(int)` | O(1) | 随机访问极快 |
| `set(int, E)` | O(1) | 直接覆盖 |
| `add(E)` | 均摊 O(1) | 末尾追加，扩容时 O(n) |
| `add(int, E)` | O(n) | 需移动后续元素 |
| `remove(int)` | O(n) | 需移动后续元素 |
| `remove(Object)` | O(n) | 先线性查找，再移动 |
| `contains(Object)` | O(n) | 线性扫描 |
| `indexOf(Object)` | O(n) | 线性扫描 |
| `clear()` | O(n) | 遍历置 null，便于 GC |
| `iterator()` | O(1) | 返回内部类 `Itr` |
| `forEach(Consumer)` | O(n) | 默认方法，Java 8+ |

#### 2.2.5 迭代器 fail-fast

```java
public E next() {
    checkForComodification();
    ...
}

final void checkForComodification() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException;
}
```

只要在迭代过程中**通过集合自身**（而非 `iterator.remove()`）修改结构就会触发 `ConcurrentModificationException`。`ConcurrentModificationException` 是一种**最佳努力**的检测，并不能保证 100% 触发并发问题。

#### 2.2.6 使用建议

- 已知元素数量时使用 `new ArrayList<>(initialCapacity)`，避免多次扩容与拷贝。
- 频繁在头/中部插入删除、且数据量大时，改用 `LinkedList`。
- 多读少写、要线程安全时考虑 `CopyOnWriteArrayList`（写时复制）。

### 2.3 LinkedList

#### 2.3.1 底层结构

`LinkedList` 基于**双向链表**，实现了 `List` 与 `Deque` 接口，因此既可以当列表也可以当队列/栈。

```java
public class LinkedList<E> extends AbstractSequentialList<E>
        implements List<E>, Deque<E>, Cloneable, java.io.Serializable {

    transient int size = 0;
    transient Node<E> first;   // 头节点
    transient Node<E> last;    // 尾节点

    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;
        Node(E item, Node<E> next, Node<E> prev) {
            this.item = item; this.next = next; this.prev = prev;
        }
    }
}
```

#### 2.3.2 关键操作源码

```java
public void addFirst(E e) {
    Node<E> f = first;
    Node<E> newNode = new Node<>(null, f, null);
    first = newNode;
    if (f == null) last = newNode;
    else f.prev = newNode;
    size++;
    modCount++;
}

public void addLast(E e) {
    Node<E> l = last;
    Node<E> newNode = new Node<>(e, null, l);
    last = newNode;
    if (l == null) first = newNode;
    else l.next = newNode;
    size++;
    modCount++;
}

public E get(int index) {
    checkElementIndex(index);
    return node(index).item;  // 这里会根据 index 决定从前往后还是从后往前遍历
}

Node<E> node(int index) {
    if (index < (size >> 1)) {       // 前半段，从前往后
        Node<E> x = first;
        for (int i = 0; i < index; i++) x = x.next;
        return x;
    } else {                          // 后半段，从后往前
        Node<E> x = last;
        for (int i = size - 1; i > index; i--) x = x.prev;
        return x;
    }
}
```

`get(int)` 会根据 index 位置选择更短的方向遍历，是个小优化。

#### 2.3.3 时间复杂度

| 方法 | 时间复杂度 | 说明 |
| --- | --- | --- |
| `addFirst / removeFirst` | O(1) | 头插/头删 |
| `addLast / removeLast` | O(1) | 尾插/尾删 |
| `add(int, E)` / `remove(int)` | O(n) | 需要先定位 |
| `get(int)` | O(n)，平均 n/4 | 通过折半遍历 |
| `contains(Object)` | O(n) | 线性扫描 |
| `peek() / poll() / push() / pop()` | O(1) | 当作栈/队列使用 |

#### 2.3.4 Deque 接口常用方法

| 队头 | 抛异常 | 返回特殊值 |
| --- | --- | --- |
| 队头取 | `getFirst()` | `peekFirst()` |
| 队头删 | `removeFirst()` | `pollFirst()` |
| 队头插 | `addFirst()` | `offerFirst()` |
| 队尾 | `getLast()` / `addLast()` / `removeLast()` | `peekLast()` / `offerLast()` / `pollLast()` |

> 推荐用 `offer` / `poll` / `peek`，因为它们通过返回值表达失败（`null` 或 `false`），不会抛异常。

#### 2.3.5 使用建议

- 频繁在头尾插删，且不需要随机访问时，使用 `LinkedList`。
- **不要把 LinkedList 当作 ArrayList 用**——`get(i)` 是 O(n)，比 `ArrayList` 慢很多，且每个节点有额外两个指针（内存开销更大、缓存局部性差）。
- 实际工程中，**优先选 `ArrayDeque` 作为栈/队列**，比 `LinkedList` 更紧凑、更快（基于数组，无包装节点）。

### 2.4 Vector

#### 2.4.1 特性

- **线程安全**：所有方法都用 `synchronized` 修饰，是 JDK 早期提供的线程安全 List。
- **扩容机制**：默认扩容为 **2 倍**（与 ArrayList 的 1.5 倍不同）。
- 性能远不如 `ArrayList` + 手动同步或并发集合，已经**基本被淘汰**。

```java
public synchronized boolean add(E e) {
    modCount++;
    ensureCapacityHelper(elementCount + 1);
    elementData[elementCount++] = e;
    return true;
}

private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + ((capacityIncrement > 0) ? capacityIncrement : oldCapacity);
    ...
}
```

#### 2.4.2 Stack

`Stack` 继承自 `Vector`，提供 LIFO 语义：

```java
public class Stack<E> extends Vector<E> {
    public E push(E item) { add(item); return item; }
    public synchronized E pop() { ... }
    public synchronized E peek() { ... }
    public boolean empty() { return size() == 0; }
    public synchronized int search(Object o) { return indexOf(o) + 1; }
}
```

> 不推荐使用 `Stack`（继承自 `Vector`，同步开销大）。推荐使用 `ArrayDeque`，效率更高。

### 2.5 List 实现类对比

| 特性 | ArrayList | LinkedList | Vector |
| --- | --- | --- | --- |
| 底层 | 动态数组 | 双向链表 | 动态数组 |
| 随机访问 | O(1) | O(n) | O(1) |
| 头插 | O(n) | O(1) | O(n) |
| 尾插 | O(1) 均摊 | O(1) | O(1) 均摊 |
| 中间插删 | O(n) | O(n)（先定位） | O(n) |
| 内存占用 | 小 | 较大（前后指针） | 小 |
| 线程安全 | 否 | 否 | 是（粗粒度同步） |
| 默认扩容 | 1.5x | — | 2x |
| 推荐度 | ★★★★★ | ★★（被 ArrayDeque 取代） | ✗（淘汰） |

### 2.6 List 选型建议

1. **随机访问多、尾部追加为主** → `ArrayList`
2. **头尾操作频繁、要实现队列/栈** → `ArrayDeque`（数组实现、更紧凑）
3. **多读少写、要线程安全** → `CopyOnWriteArrayList`
4. **高并发写多读多** → 业务层加锁 + `ArrayList`，或使用 `Collections.synchronizedList(new ArrayList<>())`
5. **明确不可变、共享只读** → `List.of(...)` / `Collections.unmodifiableList(...)`

---

## 三、Set 接口详解

`Set` 接口继承自 `Collection`，特征是**不允许重复元素**（基于 `equals` 判断），**最多一个 null**（部分实现禁止 null）。`Set` 的常用实现有 `HashSet` / `LinkedHashSet` / `TreeSet` / `EnumSet` / `CopyOnWriteArraySet`。

### 3.1 Set 接口核心 API

`Set` 没有 `List` 那种按索引访问的方法，方法定义和 `Collection` 大体一致。Java 8+ 在 `Collection` 上补充了大量默认方法：

```java
// Collection 默认方法（Set 全部继承）
default boolean removeIf(Predicate<? super E> filter);

// Stream
default Stream<E> stream();
default Stream<E> parallelStream();

// forEach
default void forEach(Consumer<? super E> action);

// 数学集合运算（需要目标 Set 实现了对应方法）
boolean containsAll(Collection<?> c);             // 子集
boolean addAll(Collection<? extends E> c);        // 并集
boolean retainAll(Collection<?> c);                // 交集
boolean removeAll(Collection<?> c);                // 差集
```

### 3.2 HashSet

#### 3.2.1 底层结构

`HashSet` 是基于 **`HashMap`** 实现的，所有元素作为 `HashMap` 的 key 存放，value 统一为一个名为 `PRESENT` 的 `Object` 常量。所以 `HashSet` 的所有特性都源自 `HashMap`。

```java
public class HashSet<E> extends AbstractSet<E>
        implements Set<E>, Cloneable, java.io.Serializable {

    private transient HashMap<E, Object> map;
    private static final Object PRESENT = new Object();

    public HashSet() { map = new HashMap<>(); }
    public HashSet(int initialCapacity) { map = new HashMap<>(initialCapacity); }
    public HashSet(int initialCapacity, float loadFactor) {
        map = new HashMap<>(initialCapacity, loadFactor);
    }

    public boolean add(E e) {
        return map.put(e, PRESENT) == null;
    }

    public boolean contains(Object o) {
        return map.containsKey(o);
    }

    public boolean remove(Object o) {
        return map.remove(o) == PRESENT;
    }

    public int size() { return map.size(); }
}
```

#### 3.2.2 关键特性

- **不允许重复**（依赖 hashCode + equals 共同决定）
- **无序**（迭代顺序取决于哈希值与桶分布）
- **允许一个 null**
- 增删改查**均摊 O(1)**（哈希足够均匀时）
- **非线程安全**

#### 3.2.3 去重原理（重要）

对象放入 `HashSet` 时：
1. 计算 `hashCode()` 得到哈希值
2. 通过哈希值定位桶（数组下标）
3. 若桶为空直接放入；不为空则逐个用 `equals()` 比较
4. 若 `equals()` 返回 true，则视为重复，丢弃新元素

> 因此**自定义对象若要正确去重，必须同时重写 `hashCode` 与 `equals`**，且契约一致：`a.equals(b) ⇒ a.hashCode() == b.hashCode()`。

#### 3.2.4 使用示例

```java
Set<String> set = new HashSet<>();
set.add("a");
set.add("b");
set.add("a");       // 不会被加入
System.out.println(set.size());  // 2

// 自定义对象去重
class User {
    String name;
    int age;
    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User)) return false;
        User u = (User) o;
        return age == u.age && Objects.equals(name, u.name);
    }
    @Override public int hashCode() { return Objects.hash(name, age); }
}
Set<User> users = new HashSet<>();
users.add(new User("tom", 18));
users.add(new User("tom", 18));  // 不会加入
```

#### 3.2.5 性能与负载因子

`HashSet` 委托 `HashMap`，其容量与负载因子由 `HashMap` 决定（默认 16，负载 0.75）。元素数量超过 `capacity * loadFactor` 时触发扩容，桶数翻倍。

### 3.3 LinkedHashSet

#### 3.3.1 底层结构

`LinkedHashSet` 继承自 `HashSet`，底层使用 **`LinkedHashMap`**。它在哈希表基础上维护了一条**双向链表**记录插入顺序（或访问顺序，构造参数控制）。

```java
public LinkedHashSet() { super(16, .75f, true); }

HashSet(int initialCapacity, float loadFactor, boolean dummy) {
    map = new LinkedHashMap<>(initialCapacity, loadFactor);
}
```

> `dummy` 参数仅用于区分与 `HashSet(int, float)` 重载，无业务含义。

#### 3.3.2 特性

- **保持插入顺序**（默认）
- 不允许重复
- 比 `HashSet` 略慢（多维护一条链表），但仍接近 O(1) 增删查
- 可用于实现 LRU（`LinkedHashMap` 的 accessOrder=true + removeEldestEntry）

#### 3.3.3 简单 LRU 缓存

```java
class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int maxSize;
    public LRUCache(int maxSize) {
        super(16, .75f, true);  // accessOrder=true
        this.maxSize = maxSize;
    }
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > maxSize;
    }
}
```

### 3.4 TreeSet

#### 3.4.1 底层结构

`TreeSet` 基于 **`TreeMap`（红黑树）** 实现，元素是 `TreeMap` 的 key，因此：

- **有序**：自然顺序或自定义 `Comparator` 顺序
- **不允许重复**
- **不允许 null**（自然顺序下，因为 `compareTo` 会 NPE）
- **增删改查 O(log n)**

```java
public class TreeSet<E> extends AbstractSet<E>
        implements NavigableSet<E>, Cloneable, java.io.Serializable {
    private transient NavigableMap<E, Object> m;
    private static final Object PRESENT = new Object();

    public TreeSet() { this(new TreeMap<>()); }
    public TreeSet(Comparator<? super E> comparator) { this(new TreeMap<>(comparator)); }
}
```

#### 3.4.2 NavigableSet 扩展 API

```java
// 范围操作
E lower(E e);       // 小于 e 的最大元素
E floor(E e);       // 小于或等于 e 的最大元素
E ceiling(E e);     // 大于或等于 e 的最小元素
E higher(E e);      // 大于 e 的最小元素

// 头尾
E first();          // 最小的
E last();           // 最大的
E pollFirst();      // 取出并移除最小
E pollLast();       // 取出并移除最大

// 子集视图
NavigableSet<E> subSet(E fromElement, boolean fromInclusive, E toElement, boolean toInclusive);
NavigableSet<E> headSet(E toElement, boolean inclusive);
NavigableSet<E> tailSet(E fromElement, boolean inclusive);

// 逆序
NavigableSet<E> descendingSet();
Iterator<E> descendingIterator();
```

#### 3.4.3 比较器

```java
// 1. 自然顺序（元素实现 Comparable）
TreeSet<Integer> ts1 = new TreeSet<>();
ts1.add(3); ts1.add(1); ts1.add(2);
// 遍历：1 2 3

// 2. 自定义比较器
TreeSet<String> ts2 = new TreeSet<>(Comparator.comparingInt(String::length));
ts2.add("aa"); ts2.add("b"); ts2.add("ccc");
// 遍历：b aa ccc（按长度升序，长度相同按 String 自然顺序）

// 3. Lambda 比较器（Java 8+）
TreeSet<User> ts3 = new TreeSet<>((a, b) -> Integer.compare(a.getAge(), b.getAge()));
```

#### 3.4.4 使用建议

- 需要**有序 + 去重**，首选 `TreeSet`
- 频繁范围查询 / 找最值：`TreeSet` 比 `ArrayList + sort + binarySearch` 方便
- 元素若用自定义对象，必须实现 `Comparable` 或在构造时传入 `Comparator`

### 3.5 EnumSet

专为枚举设计，底层是**位向量**（一个 long 存储 64 个元素），**极快且极紧凑**：

```java
enum Day { MON, TUE, WED, THU, FRI, SAT, SUN }

EnumSet<Day> weekend = EnumSet.of(Day.SAT, Day.SUN);
EnumSet<Day> weekday = EnumSet.complementOf(weekend);

for (Day d : EnumSet.range(Day.MON, Day.FRI)) { ... }
```

### 3.6 CopyOnWriteArraySet

基于 `CopyOnWriteArrayList` 实现。**写时复制**，写操作性能差但读操作极快且线程安全，适合**读多写极少**的场景（如黑白名单）。

### 3.7 Set 实现类对比

| 特性 | HashSet | LinkedHashSet | TreeSet | EnumSet | CopyOnWriteArraySet |
| --- | --- | --- | --- | --- | --- |
| 底层 | HashMap | LinkedHashMap | TreeMap（红黑树） | 位向量 | COW 数组 |
| 顺序 | 无 | 插入顺序 | 自然 / 比较器 | 枚举声明顺序 | 插入顺序 |
| 增删查 | O(1) 均摊 | O(1) 均摊 | O(log n) | O(1) | 读 O(1)、写 O(n) |
| 允许 null | 是（1 个） | 是（1 个） | 否（取决于比较器） | 否 | 是（1 个） |
| 线程安全 | 否 | 否 | 否 | 否 | 是 |
| 适用场景 | 默认去重 | 需要保留插入顺序 | 有序去重、范围查询 | 枚举集合 | 读多写少 |

---

## 四、Map 接口详解

`Map` 用于存储**键值对（key-value）**，key 不允许重复（基于 `equals`），一个 key 最多映射一个 value。`Map` 不是 `Collection` 的子接口，但提供了 `Collection` 视图（keySet、values、entrySet）以使用 Collection 的 API。

### 4.1 Map 接口核心 API

```java
// 基本 CRUD
V put(K key, V value);
V get(Object key);
V remove(Object key);
boolean containsKey(Object key);
boolean containsValue(Object value);
int size();
boolean isEmpty();
void clear();

// 批量
void putAll(Map<? extends K, ? extends V> m);

// 默认方法（Java 8+）
default V getOrDefault(Object key, V defaultValue);
default void forEach(BiConsumer<? super K, ? super V> action);
default void replaceAll(BiFunction<? super K, ? super V, ? extends V> function);
default V putIfAbsent(K key, V value);
default boolean remove(Object key, Object value);          // 同时匹配才删
default boolean replace(K key, V oldValue, V newValue);    // CAS 替换
default V compute(K key, BiFunction<? super K, ? super V, ? extends V> remappingFunction);
default V computeIfAbsent(K key, Function<? super K, ? extends V> mappingFunction);
default V computeIfPresent(K key, BiFunction<? super K, ? super V, ? extends V> remappingFunction);
default V merge(K key, V value, BiFunction<? super V, ? super V, ? extends V> remappingFunction);

// 视图（重要）
Set<K> keySet();
Collection<V> values();
Set<Map.Entry<K, V>> entrySet();
```

### 4.2 Map.Entry 接口

```java
interface Entry<K, V> {
    K getKey();
    V getValue();
    V setValue(V value);
    static <K, V> Map.Entry<K, V> entry(K k, V v);  // Java 9+
    // Java 8+ 比较器
    static <K, V extends Comparable<? super V>> Comparator<Map.Entry<K, V>> comparingByValue();
    static <K, V> Comparator<Map.Entry<K, V>> comparingByKey();
}
```

### 4.3 HashMap（JDK 8，面试核心）

#### 4.3.1 底层结构

HashMap 是基于**哈希表**实现的，采用**数组 + 链表 + 红黑树**三种结构混合：

```text
┌────────────────────┐
│   table (Node[])   │
│  ┌────┐ ┌────┐    │
│  │ [] │ │[N1]│→[N2]│→[N3]│   链表长度 < 8
│  └────┘ └────┘    │
│  ┌────┐ ┌────┐    │
│  │ [] │ │[T1]│─→[T2]│        红黑树 (Node 数 >= 8 且 table 长度 >= 64)
│  └────┘ └────┘    │
└────────────────────┘
```

关键字段：

```java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;   // 16
static final float DEFAULT_LOAD_FACTOR = 0.75f;
static final int TREEIFY_THRESHOLD = 8;                // 链表转红黑树阈值
static final int UNTREEIFY_THRESHOLD = 6;              // 红黑树退化为链表阈值
static final int MIN_TREEIFY_CAPACITY = 64;            // 树化最小表长度

transient Node<K, V>[] table;
transient int size;
int threshold;             // 扩容阈值 = capacity * loadFactor
final float loadFactor;
```

`Node` 是 `Map.Entry` 的实现：

```java
static class Node<K, V> implements Map.Entry<K, V> {
    final int hash;
    final K key;
    V value;
    Node<K, V> next;
}
```

#### 4.3.2 hash 算法

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

设计要点：
1. `key.hashCode()` 高 16 位与低 16 位异或，**让高位也参与到下标计算中**，减少哈希冲突
2. 容量是 2 的幂时，`(n - 1) & hash` 相当于取模（位运算比 `%` 快很多）

```java
static int indexFor(int h, int length) {
    return h & (length - 1);  // 等价于 h % length，length 必为 2 的幂
}
```

#### 4.3.3 put 流程

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K, V>[] tab; Node<K, V> p; int n, i;
    // 1. 表为空则初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 2. 计算下标，桶为空则直接放
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K, V> e; K k;
        // 3. 桶不为空：先比较头节点
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 4. 红黑树节点则走树插入
        else if (p instanceof TreeNode)
            e = ((TreeNode<K, V>) p).putTreeVal(this, tab, hash, key, value);
        // 5. 链表：遍历到末尾追加，必要时树化
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1)
                        treeifyBin(tab, hash);  // 链表长度 >= 8 时尝试树化
                    break;
                }
                if (e.hash == hash && ((k = e.key) == key || key.equals(k)))
                    break;
                p = e;
            }
        }
        // 6. 已存在的 key：替换值
        if (e != null) {
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);  // LinkedHashMap 用
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);   // LinkedHashMap 用
    return null;
}
```

#### 4.3.4 扩容机制（resize）

```java
final Node<K, V>[] resize() {
    Node<K, V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1;  // 阈值翻倍
    }
    else if (oldThr > 0)
        newCap = oldThr;
    else {
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float) newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY
                  ? (int) ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K, V>[] newTab = (Node<K, V>[]) new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K, V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K, V>) e).split(this, newTab, j, oldCap);
                else {
                    // 链表拆分：JDK 8 优化，避免重新计算 hash
                    Node<K, V> loHead = null, loTail = null;
                    Node<K, V> hiHead = null, hiTail = null;
                    Node<K, V> next;
                    do {
                        next = e.next;
                        // 关键：根据 hash & oldCap == 0 决定节点位置
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null) loHead = e;
                            else loTail.next = e;
                            loTail = e;
                        } else {
                            if (hiTail == null) hiHead = e;
                            else hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

要点：
1. **容量翻倍**：容量与阈值均翻倍（仍保持 2 的幂）
2. **JDK 8 链表拆分优化**：每个节点根据 `e.hash & oldCap == 0` 决定留在原下标或迁移到 `j + oldCap`，**不需要重新计算 hash**，只需 1 次位运算
3. **树拆分**：红黑树拆分为低位树和高位树，必要时退化为链表

#### 4.3.5 get 流程

```java
public V get(Object key) {
    Node<K, V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K, V> getNode(int hash, Object key) {
    Node<K, V>[] tab; Node<K, V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        // 1. 先查头节点
        if (first.hash == hash &&
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        // 2. 链表 / 树继续查找
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)
                return ((TreeNode<K, V>) first).getTreeNode(hash, key);
            do {
                if (e.hash == hash && ((k = e.key) == key || key.equals(k)))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

#### 4.3.6 为什么容量是 2 的幂

- 桶下标计算 `hash & (n - 1)` 等价于模运算，**位运算比取模快得多**
- 扩容时只需判断**一位**（`e.hash & oldCap`）就能确定节点的新位置，**避免重新计算哈希**

#### 4.3.7 为什么负载因子默认 0.75

- **太高**：冲突概率增大，链表变长，查找变慢
- **太低**：浪费空间，扩容频繁
- 0.75 是**空间与时间**的折中（泊松分布下，桶中元素数符合期望 ≤ 0.5 时冲突率低）

#### 4.3.8 树化的两个条件

```java
if (binCount >= TREEIFY_THRESHOLD - 1)  // 链表长度 >= 8
    treeifyBin(tab, hash);

if (tab.length < MIN_TREEIFY_CAPACITY)  // 表长度 < 64
    resize();   // 优先扩容而不是树化
```

> 链表长度 ≥ 8 **且** table 长度 ≥ 64 才转红黑树；否则优先扩容降低冲突。退化阈值为 6，扩容拆分时如果节点数 ≤ 6 会退化成链表。

#### 4.3.9 使用建议

- **预分配容量**：`new HashMap<>(expectedSize / 0.75f + 1)`，避免扩容
- key 必须正确实现 `hashCode` 与 `equals`
- 不允许用可变对象作为 key（修改后哈希变化导致查不到、内存泄漏）

### 4.4 LinkedHashMap

#### 4.4.1 底层结构

继承自 `HashMap`，在 `Node` 基础上扩展为 `Entry`，新增 `before` / `after` 指针维护**双向链表**，可按**插入顺序**或**访问顺序**遍历。

```java
static class Entry<K, V> extends HashMap.Node<K, V> {
    Entry<K, V> before, after;
    Entry(int hash, K key, V value, Node<K, V> next) {
        super(hash, key, value, next);
    }
}

public LinkedHashMap(int initialCapacity, float loadFactor, boolean accessOrder) {
    super(initialCapacity, loadFactor);
    this.accessOrder = accessOrder;
}
```

- `accessOrder = false`（默认）：按插入顺序
- `accessOrder = true`：按访问顺序，每次 `get` 都会把节点移到尾部（**实现 LRU 的核心**）

#### 4.4.2 LRU 实现

```java
public class LRU<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;
    public LRU(int capacity) {
        super(16, 0.75f, true);
        this.capacity = capacity;
    }
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }
}
```

### 4.5 TreeMap

#### 4.5.1 底层结构

基于**红黑树**实现，**key 有序**（自然顺序或自定义 `Comparator`），增删改查 **O(log n)**。

#### 4.5.2 NavigableMap API

```java
Map.Entry<K, V> lowerEntry(K key);
Map.Entry<K, V> floorEntry(K key);
Map.Entry<K, V> ceilingEntry(K key);
Map.Entry<K, V> higherEntry(K key);

Map.Entry<K, V> firstEntry();
Map.Entry<K, V> lastEntry();
Map.Entry<K, V> pollFirstEntry();
Map.Entry<K, V> pollLastEntry();

NavigableMap<K, V> subMap(K fromKey, boolean fromInclusive, K toKey, boolean toInclusive);
```

#### 4.5.3 使用场景

- 需要 **有序 + key 范围查询 / 找最值**：首选 TreeMap
- 子树迭代性能比遍历 HashMap 转 list + sort 略慢，但能持续维护有序

### 4.6 Hashtable

#### 4.6.1 特性

- 早期线程安全 Map，方法都用 `synchronized` 修饰
- **不允许 key / value 为 null**
- 默认容量 11，扩容 `2 * old + 1`
- 已被 `ConcurrentHashMap` 取代

### 4.7 ConcurrentHashMap（重点）

#### 4.7.1 设计目标

提供**高并发**的线程安全 Map，相比 `Hashtable` 的全表锁和 `Collections.synchronizedMap` 的单一互斥锁，CHM 通过**分段锁 / CAS** 实现更高的并发度。

#### 4.7.2 JDK 7 vs JDK 8

| 版本 | JDK 7 | JDK 8+ |
| --- | --- | --- |
| 结构 | Segment 数组 + HashEntry 数组 + 链表 | Node 数组 + 链表 / 红黑树（与 HashMap 类似） |
| 并发控制 | Segment 锁（默认 16 段） | CAS + `synchronized`（只锁桶头节点） |
| size 统计 | 多次统计取最优（避免一致性开销） | baseCount + counterCells（类似 LongAdder） |
| 扩容 | Segment 内独立扩容 | 多线程协同扩容（transfer） |

#### 4.7.3 关键字段（JDK 8+）

```java
transient volatile Node<K, V>[] table;        // 桶数组，volatile 保持可见性
private transient volatile Node<K, V>[] nextTable; // 扩容时的新表
private transient volatile int sizeCtl;       // 控制扩容标识
private transient volatile long baseCount;
private transient volatile CounterCell[] counterCells;
```

#### 4.7.4 put 流程概要

1. 表为空则 `initTable()`（CAS）
2. 桶为空则 CAS 写入新节点
3. 桶非空：
   - `MOVED` → 协助扩容（`helpTransfer`）
   - 红黑树 → 树插入
   - 链表 → `synchronized` 锁头节点后遍历插入
4. 链表节点数达到阈值 → 树化
5. 更新元素计数（`addCount`）

#### 4.7.5 get 流程

- 不需要加锁（`volatile` 保证可见性）
- 沿桶链表 / 树查找
- 弱一致性：可能读到中途正在扩容的数据，但**不会抛 `ConcurrentModificationException`**，是 **fail-safe** 迭代器

#### 4.7.6 为什么不允许 null key / value

- 多线程下 `containsKey` + `put` 组合不能保证原子，null 无法明确表示"该 key 不存在"

### 4.8 Map 实现类对比

| 特性 | HashMap | LinkedHashMap | TreeMap | Hashtable | ConcurrentHashMap |
| --- | --- | --- | --- | --- | --- |
| 底层 | 哈希表 | 哈希表 + 链表 | 红黑树 | 哈希表 | 哈希表（分段/CAS） |
| 顺序 | 无 | 插入/访问 | 自然/比较器 | 无 | 无 |
| 增删查 | O(1) 均摊 | O(1) 均摊 | O(log n) | O(1) 均摊 | O(1) 均摊 |
| 允许 null | KV 都允许 | KV 都允许 | 不允许 key | 都不允许 | 都不允许 |
| 线程安全 | 否 | 否 | 否 | 是（粗粒度） | 是（细粒度） |
| 迭代器 | fail-fast | fail-fast | fail-fast | fail-fast | fail-safe（弱一致） |
| 推荐度 | ★★★★★ | ★★★★ | ★★★★ | ✗ | ★★★★★ |

---

## 五、Queue / Deque 接口详解

`Queue` 是一种**先进先出（FIFO）** 的数据结构。`Deque`（Double Ended Queue）是双端队列，支持在两端插入和删除元素。

### 5.1 Queue 接口方法分组

| 行为 | 抛异常 | 返回特殊值 |
| --- | --- | --- |
| 队尾插入 | `add(e)` | `offer(e)`（推荐） |
| 队头取并删除 | `remove()` | `poll()`（推荐） |
| 队头取不删 | `element()` | `peek()`（推荐） |

### 5.2 Deque 接口方法

| 位置 | 抛异常 | 返回特殊值 |
| --- | --- | --- |
| 头插 | `addFirst(e)` | `offerFirst(e)` |
| 尾插 | `addLast(e)` | `offerLast(e)` |
| 头取删 | `removeFirst()` | `pollFirst()` |
| 尾取删 | `removeLast()` | `pollLast()` |
| 头查看 | `getFirst()` | `peekFirst()` |
| 尾查看 | `getLast()` | `peekLast()` |
| 栈压入 | `push(e)`（= addFirst） | |
| 栈弹出 | `pop()`（= removeFirst） | |

### 5.3 ArrayDeque

#### 5.3.1 底层结构

基于**循环数组**实现的双端队列，可作为**栈**（比 `Stack` 快）或**队列**（比 `LinkedList` 紧凑）。

```java
public class ArrayDeque<E> extends AbstractCollection<E>
        implements Deque<E>, Cloneable, Serializable {

    transient Object[] elements;
    transient int head;
    transient int tail;
    private static final int MIN_INITIAL_CAPACITY = 8;
}
```

#### 5.3.2 关键特性

- 容量始终为 2 的幂
- `head` 与 `tail` 在数组两端向中间扩展，到达边界时循环
- 扩容时翻倍，并重新映射头尾位置
- **不允许 null 元素**

#### 5.3.3 用法

```java
// 当作栈
Deque<Integer> stack = new ArrayDeque<>();
stack.push(1);
stack.push(2);
stack.push(3);
while (!stack.isEmpty()) {
    System.out.println(stack.pop());  // 3 2 1
}

// 当作队列
Deque<Integer> queue = new ArrayDeque<>();
queue.offer(1);
queue.offer(2);
queue.offer(3);
while (!queue.isEmpty()) {
    System.out.println(queue.poll());  // 1 2 3
}
```

### 5.4 PriorityQueue

#### 5.4.1 底层结构

基于**二叉小顶堆**（数组实现）实现的优先级队列。**默认自然顺序**（最小堆），可通过 `Comparator` 自定义。

```java
public class PriorityQueue<E> extends AbstractQueue<E>
        implements java.io.Serializable {

    private static final int DEFAULT_INITIAL_CAPACITY = 11;
    transient Object[] queue;
    int size;
    private final Comparator<? super E> comparator;
}
```

#### 5.4.2 堆操作

```java
// 上浮：插入时保证堆序
private void siftUpComparable(int k, E x) {
    Comparable<? super E> key = (Comparable<? super E>) x;
    while (k > 0) {
        int parent = (k - 1) >>> 1;
        Object e = queue[parent];
        if (key.compareTo((E) e) >= 0) break;
        queue[k] = e;
        k = parent;
    }
    queue[k] = key;
}

// 下沉：删除堆顶时维持堆序
private void siftDownComparable(int k, E x) {
    int half = size >>> 1;
    while (k < half) {
        int child = (k << 1) + 1;
        Object c = queue[child];
        int right = child + 1;
        if (right < size && ((Comparable<? super E>) c).compareTo((E) queue[right]) > 0)
            c = queue[child = right];
        if (key.compareTo((E) c) <= 0) break;
        queue[k] = c;
        k = child;
    }
    queue[k] = key;
}
```

#### 5.4.3 时间复杂度

| 操作 | 时间复杂度 |
| --- | --- |
| `offer` | O(log n) |
| `poll` | O(log n) |
| `peek` | O(1) |
| `remove(Object)` | O(n) |
| `add` | O(log n) |

#### 5.4.4 使用建议

- 默认小顶堆：每次 `poll` 取出最小元素
- 大顶堆：传入 `Comparator.reverseOrder()` 或 `Collections.reverseOrder()`
- **不允许 null 元素**
- **非线程安全**，需要线程安全请用 `PriorityBlockingQueue`

### 5.5 BlockingQueue（并发队列入口）

`java.util.concurrent.BlockingQueue` 增加了阻塞能力，是生产者-消费者模式的经典工具：

| 实现类 | 数据结构 | 容量 | 特点 |
| --- | --- | --- | --- |
| `ArrayBlockingQueue` | 数组 | 有界（必填） | 单锁实现 |
| `LinkedBlockingQueue` | 链表 | 可选有界（默认 Integer.MAX_VALUE） | 双锁（take/put 各一锁） |
| `PriorityBlockingQueue` | 堆 | 无界 | 元素按优先级 |
| `SynchronousQueue` | 不存储元素 | 0 | 插入必须等待取出 |
| `DelayQueue` | 堆 | 无界 | 元素到期才能取出 |
| `LinkedTransferQueue` | 链表 | 无界 | `transfer` 直接交付给消费者 |

```java
BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(10);
queue.put(1);     // 满则阻塞
Integer x = queue.take();  // 空则阻塞
```

---

## 六、集合工具类

### 6.1 Collections

`Collections` 提供了一组操作集合的**静态方法**。

#### 6.1.1 排序与查找

```java
// 排序
Collections.sort(list);                          // 自然顺序
Collections.sort(list, comparator);              // 自定义比较器
Collections.reverse(list);                       // 倒序
Collections.shuffle(list);                       // 随机打乱
Collections.swap(list, i, j);                    // 交换
Collections.rotate(list, distance);               // 旋转

// 二分查找（要求已排序）
int idx = Collections.binarySearch(list, key);
int idx = Collections.binarySearch(list, key, comparator);

// 最大最小值
T max = Collections.max(coll);
T min = Collections.min(coll);
```

#### 6.1.2 同步包装（装饰器模式）

```java
List<T> syncList = Collections.synchronizedList(new ArrayList<>());
Set<T> syncSet  = Collections.synchronizedSet(new HashSet<>());
Map<K,V> syncMap = Collections.synchronizedMap(new HashMap<>());

// 用 synchronized 块包裹迭代
synchronized (syncList) {
    Iterator<T> it = syncList.iterator();
    while (it.hasNext()) { ... }
}
```

#### 6.1.3 不可变包装

```java
List<T> unmodList = Collections.unmodifiableList(list);
Set<T> unmodSet  = Collections.unmodifiableSet(set);
Map<K,V> unmodMap = Collections.unmodifiableMap(map);

// 直接生成不可变集合
List<String> emptyList = Collections.emptyList();
Set<String> singletonSet = Collections.singleton("x");
List<String> nCopies = Collections.nCopies(10, "x");
```

> 不可变集合对任何修改操作都会抛 `UnsupportedOperationException`。JDK 9+ 推荐用 `List.of(...)` / `Map.of(...)` / `Set.of(...)`。

#### 6.1.4 单元素集合

```java
List<String> single = Collections.singletonList("only");
Set<String> singleSet = Collections.singleton("only");
Map<String, Integer> singleMap = Collections.singletonMap("k", 1);
```

#### 6.1.5 类型检查

```java
Collections.checkedList(list, String.class);  // 防止绕过泛型插入错误类型
```

### 6.2 Arrays

#### 6.2.1 数组与 List 互转

```java
List<String> list = Arrays.asList("a", "b", "c");  // 固定大小 List
String[] arr = list.toArray(new String[0]);
```

> `Arrays.asList(...)` 返回的是 `Arrays.ArrayList`（内部类），**不能 add / remove**，否则抛 `UnsupportedOperationException`。想要可变 List 用 `new ArrayList<>(Arrays.asList(...))`。

#### 6.2.2 排序与搜索

```java
Arrays.sort(arr);                          // 自然顺序
Arrays.sort(arr, fromIndex, toIndex);      // 部分排序
Arrays.sort(arr, comparator);              // 自定义
Arrays.parallelSort(arr);                  // 并行排序（JDK 8+，大数据量时性能更好）

int idx = Arrays.binarySearch(arr, key);
```

#### 6.2.3 填充 / 比较 / 复制

```java
Arrays.fill(arr, 0);                       // 全填
Arrays.fill(arr, fromIndex, toIndex, 0);   // 部分填

boolean eq = Arrays.equals(arr1, arr2);    // 元素相等
boolean deepEq = Arrays.deepEquals(o1, o2);// 多维数组

int[] copy = Arrays.copyOf(arr, newLength);
int[] range = Arrays.copyOfRange(arr, from, to);
```

#### 6.2.4 JDK 8+ Stream 互转

```java
IntStream stream = Arrays.stream(arr);
String[] arr = stream.toArray(String[]::new);

List<String> list = Arrays.stream(arr).collect(Collectors.toList());
```

#### 6.2.5 JDK 9+ 便捷方法

```java
int[] arr = { 3, 1, 4, 1, 5 };
int lo = Arrays.stream(arr).min().getAsInt();
int sum = Arrays.stream(arr).sum();
int[] sorted = Arrays.stream(arr).sorted().toArray();
boolean anyGt4 = Arrays.stream(arr).anyMatch(x -> x > 4);
```

### 6.3 不可变集合工厂（JDK 9+）

```java
List<String> list = List.of("a", "b", "c");        // 不允许 null
Set<Integer> set = Set.of(1, 2, 3);
Map<String, Integer> map = Map.of("a", 1, "b", 2);
Map<String, Integer> m2 = Map.ofEntries(
    Map.entry("a", 1),
    Map.entry("b", 2)
);

// JDK 10+ 提供 copyOf
List<String> copy = List.copyOf(list);              // null 不允许
```

不可变集合优势：
- 节省内存（紧凑表示）
- 线程安全
- 防止误修改

---

## 七、Java 8 Stream API

Stream 是对集合（Collection / 数组）的**函数式操作抽象**，让链式转换、过滤、聚合、并行化变得简洁。**Stream 不存储数据、不修改源集合**（除非明确终止操作设计如此）。

### 7.1 创建 Stream

```java
// 从集合
list.stream();                 // 顺序流
list.parallelStream();         // 并行流

// 从数组
Arrays.stream(arr);
Stream.of("a", "b", "c");     // Stream.of 静态方法

// 从文件 / 网络
Files.lines(Paths.get("a.txt"), StandardCharsets.UTF_8);

// 数值流（避免装箱）
IntStream.range(0, 100);
IntStream.rangeClosed(0, 100);
LongStream, DoubleStream 类似

// 构建器
Stream.Builder<String> b = Stream.builder();
b.add("a").add("b");
Stream<String> s = b.build();
```

### 7.2 中间操作（Intermediate）

**惰性求值**：不会立即执行，直到终止操作触发。

| 操作 | 说明 |
| --- | --- |
| `filter(Predicate)` | 过滤元素 |
| `map(Function)` | 一对一转换 |
| `mapToInt / mapToLong / mapToDouble` | 转数值流 |
| `flatMap(Function)` | 一对多展平 |
| `distinct()` | 去重（依赖 equals） |
| `sorted()` / `sorted(Comparator)` | 排序 |
| `peek(Consumer)` | 调试用消费，不改变元素 |
| `limit(n)` | 截取前 n |
| `skip(n)` | 跳过前 n |
| `takeWhile(Predicate)` | JDK 9+ 满足条件时取 |
| `dropWhile(Predicate)` | JDK 9+ 满足条件时丢弃 |

### 7.3 终止操作（Terminal）

**触发整个流水线的执行**。

#### 7.3.1 遍历与匹配

```java
forEach(Consumer);
forEachOrdered(Consumer);  // 并行流中保证顺序
boolean anyMatch(Predicate);
boolean allMatch(Predicate);
boolean noneMatch(Predicate);
```

#### 7.3.2 聚合

```java
long count();
Optional<T> min(Comparator);
Optional<T> max(Comparator);
```

#### 7.3.3 归约

```java
Optional<T> reduce(BinaryOperator<T> accumulator);
T reduce(T identity, BinaryOperator<T> accumulator);   // 含初始值
<U> U reduce(U identity, BiFunction<U, ? super T, U> accumulator, BinaryOperator<U> combiner);
```

#### 7.3.4 收集（Collectors）

```java
// 转集合
List<T> list = stream.collect(Collectors.toList());
Set<T> set = stream.collect(Collectors.toSet());
Map<K, T> map = stream.collect(Collectors.toMap(t -> t.getKey(), t -> t));

// 拼接
String s = stream.collect(Collectors.joining(", ", "[", "]"));

// 分组
Map<K, List<T>> groups = stream.collect(Collectors.groupingBy(t -> t.getCategory()));
Map<K, Long> counts = stream.collect(Collectors.groupingBy(t -> t.getCategory(), Collectors.counting()));

// 分区
Map<Boolean, List<T>> parts = stream.collect(Collectors.partitioningBy(t -> t.isActive()));

// 汇总（数值）
DoubleSummaryStatistics stats = stream.collect(Collectors.summarizingDouble(T::getScore));
// sum / avg / min / max / count

// 自定义收集器
Collector.of(supplier, accumulator, combiner, finisher);
```

### 7.4 数值流

```java
IntStream is = IntStream.rangeClosed(1, 100);
int sum = is.sum();                       // 5050
OptionalDouble avg = is.average();
IntSummaryStatistics stat = is.summaryStatistics();
```

数值流提供 sum / average / max / min / range 等方法，避免装箱开销。

### 7.5 并行流

```java
list.parallelStream().filter(x -> x > 0).forEach(System.out::println);

// 顺序保持
list.parallelStream().forEachOrdered(...);
```

> **注意**：并行流底层是 `ForkJoinPool.commonPool()`。盲目使用反而更慢（线程切换开销），仅在数据量大、计算密集、源集合易切分（`ArrayList` / `数组`）时考虑。

### 7.6 Stream 完整示例

```java
List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "Dave");

// 找出长度 > 3 的名字，按长度升序，转大写，用逗号拼接
String result = names.stream()
    .filter(n -> n.length() > 3)
    .sorted(Comparator.comparingInt(String::length))
    .map(String::toUpperCase)
    .collect(Collectors.joining(", "));
// "ALICE, CHARLIE, DAVE"
```

### 7.7 Optional

Stream 常与 `Optional` 搭配：

```java
Optional<String> opt = list.stream().findFirst();
opt.ifPresent(System.out::println);
String v = opt.orElse("default");
String v2 = opt.orElseGet(() -> "lazy default");
String v3 = opt.orElseThrow(() -> new IllegalStateException("empty"));
```

---

## 八、并发集合（java.util.concurrent）

### 8.1 整体分类

```text
Concurrent
├── ConcurrentMap
│   └── ConcurrentHashMap
├── ConcurrentQueue
│   ├── ConcurrentLinkedQueue   (无锁非阻塞)
│   └── ConcurrentLinkedDeque
├── BlockingQueue
│   ├── ArrayBlockingQueue
│   ├── LinkedBlockingQueue
│   ├── PriorityBlockingQueue
│   ├── SynchronousQueue
│   ├── DelayQueue
│   └── LinkedTransferQueue
├── CopyOnWrite
│   ├── CopyOnWriteArrayList
│   └── CopyOnWriteArraySet
└── 并发 List 接口包装
    └── Collections.synchronizedList
```

### 8.2 CopyOnWriteArrayList

#### 8.2.1 原理

写操作时把原数组**整体复制一份**，在新数组上修改，然后用 CAS 替换引用。读操作**完全无锁**。

```java
public boolean add(E e) {
    synchronized (lock) {
        Object[] es = getArray();
        int len = es.length;
        es = Arrays.copyOf(es, len + 1);
        es[len] = e;
        setArray(es);
        return true;
    }
}

private E get(Object[] a, int index) {
    return (E) a[index];
}
```

#### 8.2.2 特性

- **读极快、写极慢**（O(n) 拷贝）
- **线程安全**
- 迭代器是 **快照式**，遍历时数据不变 → fail-safe
- 适合**读多写极少**、监听器列表、白名单等

#### 8.2.3 CopyOnWriteArraySet

基于 `CopyOnWriteArrayList`，特性类似，但内部去重需要遍历比对，开销更大。一般并发场景优先用 `ConcurrentHashMap.newKeySet()` 替代。

### 8.3 ConcurrentLinkedQueue

无锁（CAS）的**非阻塞线程安全队列**，基于 Michael & Scott 算法实现的无界链表。`size()` 较慢（需要遍历），`offer` / `poll` 是 O(1)。

### 8.4 BlockingQueue 用法

#### 8.4.1 核心方法

| 行为 | 抛异常 | 返回特殊值 | 阻塞 | 超时 |
| --- | --- | --- | --- | --- |
| 插入 | `add(e)` | `offer(e)` | `put(e)` | `offer(e, timeout, unit)` |
| 取出 | `remove()` | `poll()` | `take()` | `poll(timeout, unit)` |
| 检查 | `element()` | `peek()` | — | — |

#### 8.4.2 生产者-消费者模板

```java
BlockingQueue<Task> queue = new LinkedBlockingQueue<>(1000);

// 生产者
new Thread(() -> {
    while (running) {
        Task t = produce();
        try { queue.put(t); } catch (InterruptedException e) { Thread.currentThread().interrupt(); break; }
    }
}).start();

// 消费者
new Thread(() -> {
    while (running) {
        try {
            Task t = queue.take();
            handle(t);
        } catch (InterruptedException e) { Thread.currentThread().interrupt(); break; }
    }
}).start();
```

#### 8.4.3 DelayQueue

元素实现 `Delayed.getDelay(TimeUnit)`，**到期才能取出**，常用于定时任务、缓存过期。

```java
class DelayedTask implements Delayed {
    private final long deadline;
    public long getDelay(TimeUnit unit) { return unit.convert(deadline - System.nanoTime(), NANOSECONDS); }
    public int compareTo(Delayed o) { return Long.compare(this.deadline, ((DelayedTask) o).deadline); }
}
```

#### 8.4.4 SynchronousQueue

容量为 0，每个插入必须等待对应取出。常用于线程间直接交付任务。

### 8.5 ConcurrentSkipListMap / ConcurrentSkipListSet

基于**跳表（SkipList）** 的并发有序 Map / Set，提供 O(log n) 的增删查，并发友好（无锁化分段 CAS）。

适用场景：需要并发 + 有序 + 范围查询。

### 8.6 集合并发方案速查

| 场景 | 推荐 |
| --- | --- |
| 高并发 KV 读写 | `ConcurrentHashMap` |
| 并发有序 KV | `ConcurrentSkipListMap` |
| 高并发读多写少 List | `CopyOnWriteArrayList` |
| 并发 Set | `ConcurrentHashMap.newKeySet()` |
| 生产者-消费者 | `LinkedBlockingQueue` / `ArrayBlockingQueue` |
| 定时 / 延迟任务 | `DelayQueue` / `ScheduledExecutorService` |
| 线程间直接交换数据 | `SynchronousQueue` / `Exchanger` |
| 单线程 | 不用并发行列，会拖慢性能 |

---

## 九、选型指南（一图速查）

```text
需要存什么？
├── 单列元素
│   ├── 允许重复 + 有序
│   │   ├── 随机访问多 → ArrayList
│   │   ├── 头尾操作多 → ArrayDeque
│   │   └── 高并发读少写 → CopyOnWriteArrayList
│   ├── 不可重复
│   │   ├── 无序 → HashSet
│   │   ├── 插入顺序 → LinkedHashSet
│   │   ├── 自然/比较器顺序 → TreeSet
│   │   ├── 枚举 → EnumSet
│   │   └── 高并发 → ConcurrentHashMap.newKeySet()
│   ├── FIFO 队列
│   │   ├── 单线程 → ArrayDeque
│   │   ├── 多线程无阻塞 → ConcurrentLinkedQueue
│   │   └── 多线程阻塞 → LinkedBlockingQueue / ArrayBlockingQueue
│   ├── LIFO 栈 → ArrayDeque
│   └── 优先级 → PriorityQueue / PriorityBlockingQueue
│
└── 键值对
    ├── 单线程
    │   ├── 无序 → HashMap
    │   ├── 插入顺序 → LinkedHashMap
    │   ├── LRU → LinkedHashMap(accessOrder=true)
    │   └── 有序 → TreeMap
    ├── 多线程 → ConcurrentHashMap
    ├── 多线程有序 → ConcurrentSkipListMap
    └── 已被淘汰 → Hashtable / Collections.synchronizedMap
```

### 9.1 通用最佳实践

1. **面向接口编程**：变量声明为 `List` / `Map`，按需选实现
2. **预分配容量**：构造时给出预期大小，避免扩容
3. **正确实现 hashCode / equals**：自定义 key / Set 元素必备
4. **不可变对象做 key**：String / Integer / Long 等是好的 key
5. **避免在并发场景使用非线程安全集合**：尤其注意 HashMap / ArrayList 在多线程下可能死循环（扩容链表环）、数据丢失
6. **迭代时不要结构性修改**：需要则用 `Iterator.remove()` 或 `ListIterator`
7. **优先使用 `ArrayDeque` 替代 `Stack` 和 `LinkedList`** 作为栈/队列
8. **优先使用 JDK 9+ `List.of(...)` / `Map.of(...)`** 表达不可变集合
9. **大数据 + 简单操作考虑 Stream**，但避免在循环中频繁创建 Stream
10. **并行流谨慎使用**，先 benchmark

### 9.2 常见容量规划公式

```java
// Guava 的 Maps.newHashMapWithExpectedSize 同款
int expected = 1000;
int cap = (int) (expected / 0.75f + 1);
Map<K, V> map = new HashMap<>(cap);
```

---

## 十、高频面试题汇总

### 10.1 ArrayList vs LinkedList

| 维度 | ArrayList | LinkedList |
| --- | --- | --- |
| 底层 | 动态数组 | 双向链表 |
| 随机访问 | O(1) | O(n) |
| 头插 | O(n) | O(1) |
| 尾插 | 均摊 O(1) | O(1) |
| 中间插删 | O(n) | O(n)（定位 + 改指针） |
| 内存占用 | 紧凑 | 大（前后指针 + 包装对象） |
| 缓存局部性 | 优 | 差 |

实际工程中**ArrayList 在绝大多数场景下更优**，因为 CPU 缓存命中率高、GC 压力小、连续内存访问快。

### 10.2 ArrayList 扩容机制

- 初始空数组，首次 add 才扩容到 10
- 每次扩容 1.5 倍（`oldCapacity + oldCapacity >> 1`）
- 通过 `Arrays.copyOf` 复制，开销是 O(n)

### 10.3 HashMap put 流程

1. 计算 key 的 hash（高 16 位异或低 16 位）
2. 若 table 为空则 resize
3. 计算桶下标 `i = (n - 1) & hash`
4. 桶为空 → 直接放
5. 桶非空：
   - 头节点匹配 → 覆盖
   - 红黑树节点 → 树插入
   - 链表 → 遍历到末尾追加；长度 ≥ 8 且 table 长度 ≥ 64 时树化
6. size > threshold → resize

### 10.4 HashMap 树化的两个条件

- 链表长度 ≥ `TREEIFY_THRESHOLD` (8)
- table 长度 ≥ `MIN_TREEIFY_CAPACITY` (64)

否则优先扩容而不是树化。

### 10.5 HashMap 为什么线程不安全

- 扩容时多线程并发可能导致**链表环**（JDK 7 的 transfer 用头插法；JDK 8 改用尾插法降低了概率，但仍不保证）
- put 时可能**值覆盖**（非原子）

解决方案：使用 `ConcurrentHashMap` 或 `Collections.synchronizedMap`。

### 10.6 ConcurrentHashMap 1.7 vs 1.8

| 版本 | 数据结构 | 并发控制 |
| --- | --- | --- |
| 1.7 | Segment + HashEntry + 链表 | Segment 锁（ReentrantLock） |
| 1.8 | Node + 链表/红黑树 | CAS + `synchronized` 锁桶头节点 |

JDK 8 简化了实现、减少了锁粒度、引入红黑树优化极端冲突场景。

### 10.7 为什么 ConcurrentHashMap 不允许 null key/value

`containsKey` + `put` 在并发下不能保证原子，若允许 null，无法区分"key 不存在返回 null"与"key 存在但 value 是 null"。

### 10.8 HashSet 怎么保证元素唯一

底层是 HashMap，元素作为 key、value 是固定常量 `PRESENT`。`add(e)` 通过 `map.put(e, PRESENT) == null` 判断是否新增。

### 10.9 fail-fast vs fail-safe

| 机制 | 触发 | 表现 | 典型代表 |
| --- | --- | --- | --- |
| fail-fast | 迭代时检测到 modCount 变化 | 抛 `ConcurrentModificationException` | ArrayList / HashMap / HashSet |
| fail-safe | 迭代器在修改前的快照上工作 | 不抛异常，但数据弱一致 | CopyOnWriteArrayList / ConcurrentHashMap |

### 10.10 Collection 与 Collections 区别

- `java.util.Collection`：**接口**，集合框架的根
- `java.util.Collections`：**工具类**，提供静态方法

### 10.11 Iterator 与 ListIterator

| 特性 | Iterator | ListIterator |
| --- | --- | --- |
| 适用 | 所有 Collection | 仅 List |
| 遍历方向 | 单向 | 双向 |
| 修改 | `remove()` | `remove() / set() / add()` |
| 索引 | 不支持 | `nextIndex() / previousIndex()` |

### 10.12 Arrays.asList 的坑

- 返回 `Arrays.ArrayList`（内部类），**固定大小**，不能 add/remove
- 修改数组会影响 List（视图）
- 期望可变用 `new ArrayList<>(Arrays.asList(...))`

### 10.13 equals 与 hashCode 契约

- **equals 相等** ⇒ **hashCode 相等**
- hashCode 相等不一定 equals 相等（哈希冲突）
- 自定义对象作为 HashMap key / HashSet 元素**必须同时重写两者**，且参与 equals 比较的字段必须参与 hashCode 计算

### 10.14 HashMap 与 Hashtable 区别

| 维度 | HashMap | Hashtable |
| --- | --- | --- |
| 线程安全 | 否 | 是（全方法 synchronized） |
| null | 允许 1 个 null key、多个 null value | 都不允许 |
| 性能 | 高 | 低 |
| 推荐 | 是 | 已淘汰 |

### 10.15 Comparable vs Comparator

| 维度 | Comparable | Comparator |
| --- | --- | --- |
| 位置 | 类内部实现 | 外部独立类 / Lambda |
| 方法 | `compareTo(T)` | `compare(T, T)` |
| 数量 | 一个自然顺序 | 多个独立策略 |
| Collections.sort | sort(list) | sort(list, comparator) |

### 10.16 哪些集合是线程安全的

- 旧：`Vector` / `Stack` / `Hashtable` / `Enumeration` 相关
- 工具包装：`Collections.synchronizedXxx`
- 并发包：`ConcurrentHashMap` / `ConcurrentSkipListMap` / `ConcurrentLinkedQueue` / `CopyOnWriteArrayList` 等

### 10.17 集合的快速失败（fail-fast）原理

迭代器内部维护 `expectedModCount`，每次 `next/remove` 前检查 `modCount != expectedModCount`，若不等则抛 `ConcurrentModificationException`。

```java
final void checkForComodification() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
}
```

### 10.18 优先级队列底层

二叉小顶堆（数组），父子通过下标计算：
- 左子 = `2*i + 1`
- 右子 = `2*i + 2`
- 父 = `(i - 1) >> 1`

### 10.19 BlockingQueue 核心方法

| 行为 | 抛异常 | 特殊值 | 阻塞 | 超时 |
| --- | --- | --- | --- | --- |
| 插入 | add | offer | put | offer(..., t, u) |
| 取出 | remove | poll | take | poll(..., t, u) |
| 检查 | element | peek | — | — |

### 10.20 ConcurrentHashMap size 实现

类似 `LongAdder`，用 `baseCount` + `CounterCell[]`。每个线程优先更新自己槽位的 `CounterCell.value`，减少竞争，最后 sum 得到总数。

---

## 十一、补充：Java 9~17 的新特性

### 11.1 JDK 9

- `List.of(...)` / `Set.of(...)` / `Map.of(...)` / `Map.ofEntries(...)` 不可变集合工厂
- `Stream.takeWhile` / `dropWhile` / `ofNullable`
- `Optional.stream()`

### 11.2 JDK 10

- `List.copyOf(...)` / `Map.copyOf(...)` / `Set.copyOf(...)`
- 局部变量类型推断 `var`

### 11.3 JDK 11

- `Collection.toArray(IntFunction)` 默认方法
- 集合 API 增强（如 `Set.of` 已有接口签名）

### 11.4 JDK 17

- Sealed Classes 增强模式匹配
- 模式匹配 switch（预览）

> 这些特性在日常开发中可按需使用，最关键的还是集合本身的设计与原理。

---

## 十二、参考资料

- 《Effective Java》第 3 版 — Joshua Bloch（条目 45~58 集合与 lambda 最佳实践）
- 《Java 核心技术》第 12 版 — Cay S. Horstmann
- 《深入理解 Java 虚拟机》 — 周志明
- OpenJDK 源码 `java.util` / `java.util.concurrent`
- Java 官方文档：<https://docs.oracle.com/en/java/javase/>

---

> **本文目标**：覆盖 Java 后端面试与日常开发中 90% 以上的集合知识。建议先理解各集合的**底层数据结构**与**复杂度**，再去记忆 API 与最佳实践。











