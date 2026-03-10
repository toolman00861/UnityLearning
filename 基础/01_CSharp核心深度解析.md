# 01. C# 核心深度解析：内存、性能与底层

## 1. 值类型与引用类型：内存视角的本质区别

在 Unity 开发中，理解内存布局是优化 GC（垃圾回收）和提升性能的基石。

### 1.1 存储位置与分配机制
*   **值类型 (Value Type)**：`int`, `float`, `struct`, `bool`, `enum`
    *   **存储**：通常存储在**栈 (Stack)** 上（作为局部变量时），或内联存储在包含它的引用类型中（作为类字段时）。
    *   **分配**：速度极快，无需 GC 扫描。
    *   **赋值**：复制整个值。
*   **引用类型 (Reference Type)**：`class`, `interface`, `delegate`, `string`, `object`, `Array`
    *   **存储**：实例数据存储在**托管堆 (Managed Heap)** 上，栈上只保存一个指向堆内存的地址（指针）。
    *   **分配**：通过 `new` 关键字在堆上分配，速度较慢，受 GC 管理。
    *   **赋值**：只复制引用（指针），指向同一个对象。

### 1.2 结构体 (struct) vs 类 (class) 的选择
| 特性 | struct (值类型) | class (引用类型) |
| :--- | :--- | :--- |
| **内存开销** | 无对象头开销，紧凑 | 每个对象有头部开销 (SyncBlockIndex + MethodTablePtr)，至少 12-16 字节 |
| **GC 压力** | 0 (除非装箱) | 产生 GC Alloc，需要回收 |
| **传参成本** | 复制整个结构体 (大对象慢) | 仅复制 4/8 字节指针 (快) |
| **适用场景** | 短生命周期、数据量小 (<16字节)、不可变数据 (Vector3, Color) | 逻辑复杂、生命周期长、需要继承 (Manager, Enemy) |

**Unity 实战建议**：
*   不要为了“避免 GC”盲目把所有东西改成 `struct`。如果 `struct` 过大且频繁传递，数据拷贝的 CPU 开销可能比 GC 更严重。可以使用 `ref` 传递来优化。
*   `List<struct>` 比 `List<class>` 对 CPU 缓存更友好（内存连续）。

---

## 2. 装箱 (Boxing) 与拆箱 (Unboxing)

这是面试和性能优化中最老生常谈但也最容易被忽视的点。

### 2.1 底层过程
*   **装箱**：`IL: box`
    1.  在托管堆中分配内存（包含对象头）。
    2.  将值类型的数据复制到新分配的堆内存中。
    3.  返回堆上对象的地址。
    *   **代价**：堆内存分配 + 内存拷贝 + GC 压力。
*   **拆箱**：`IL: unbox`
    1.  检查引用是否为 null。
    2.  检查引用指向的对象是否是目标值类型的装箱实例（类型安全检查）。
    3.  返回指向对象内部值的指针（注意：unbox指令本身返回指针，通常紧接着 `ldobj` 复制值）。
    *   **代价**：类型检查 + 内存拷贝。

### 2.2 隐式装箱陷阱
```csharp
int id = 100;
object obj = id; // 显式装箱

// 陷阱 1: 字符串拼接
Debug.Log("ID: " + id); // 这里的 id 会被装箱！因为调用了 string.Concat(object, object)
// 优化: id.ToString() (无装箱，但生成 string) 或 ZString (零分配库)

// 陷阱 2: 格式化字符串
string.Format("ID: {0}", id); // 参数是 params object[]，导致装箱
// 优化: 使用 StringBuilder 或 C# 10 的 string interpolation handler (Unity 2021+ 支持)

// 陷阱 3: 接口调用
interface IReset { void Reset(); }
struct MyStruct : IReset { public void Reset() {} }

MyStruct s = new MyStruct();
s.Reset(); // 正常调用，无装箱
((IReset)s).Reset(); // 必须装箱！因为接口调用需要对象引用
```

---

## 3. 垃圾回收 (GC) 机制深度剖析

Unity (Mono/IL2CPP) 使用的是 **Boehm-Demers-Weiser (BDW) GC**（非分代，或实验性分代）或 **Incremental GC (增量式 GC)**。

### 3.1 GC 触发时机
1.  **堆内存不足**：尝试分配新对象但空间不够时。
2.  **手动调用**：`GC.Collect()`（尽量避免）。
3.  **系统低内存**：操作系统通知内存吃紧。

### 3.2 标记-清除 (Mark-Sweep) 算法
1.  **暂停 (Stop The World)**：为了数据安全，暂停所有应用线程（造成卡顿）。
2.  **标记 (Mark)**：从 **GC Roots**（静态变量、栈上变量、寄存器）出发，遍历所有引用的对象，打上“存活”标记。
3.  **清除 (Sweep)**：遍历整个堆，回收未被标记的内存块，将其加入空闲列表 (Free List)。
4.  **压缩 (Compact)**：*Unity 的默认 GC 通常不压缩*，这意味着会产生**内存碎片**。

### 3.3 增量式 GC (Incremental GC)
*   **原理**：将繁重的“标记”阶段拆分成多个小切片，分散在多帧执行。
*   **优点**：大幅减少单帧 GC 峰值耗时（解决掉帧卡顿）。
*   **缺点**：总的 GC CPU 开销略有增加；如果内存分配速度过快，GC 追不上，仍会触发 Stop The World。
*   **使用**：Project Settings -> Player -> Other Settings -> Use Incremental GC (默认开启)。

### 3.4 优化策略
*   **对象池 (Object Pooling)**：复用对象，减少 `new` 和 `Destroy`。
*   **避免闭包**：Lambda 表达式引用外部变量会生成临时类实例。
*   **缓存组件引用**：`GetComponent` 有开销且可能产生少量垃圾（视版本而定），在 `Start` 中缓存。
*   **字符串优化**：字符串是不可变的，每次修改都会生成新字符串。使用 `StringBuilder`。

---

## 4. 字符串 (String) 的不可变性

在 C# 中，`string` 是引用类型，但具有**不可变性 (Immutability)**。

### 4.1 为什么不可变？
*   **线程安全**：多线程读取无需加锁。
*   **字符串驻留池 (Intern Pool)**：字面量字符串（如 `"Hello"`）在编译期会被放入池中，全局共享一份内存。

### 4.2 性能灾难
```csharp
string s = "";
for (int i = 0; i < 1000; i++) {
    s += i; // 灾难！每次 += 都会：
            // 1. 分配新内存 (len(s) + len(i))
            // 2. 复制旧 s
            // 3. 复制 i
            // 4. 旧 s 变成垃圾
}
```
**优化**：使用 `System.Text.StringBuilder`。它在内部维护一个字符数组，修改时只操作数组，不分配新对象（除非扩容）。

---

## 5. 扩展：Unsafe 与 Stackalloc

在极致性能优化场景（如寻路、物理运算），可以使用 `unsafe` 代码块绕过类型安全检查。

*   **指针操作**：直接操作内存地址，无数组边界检查开销。
*   **stackalloc**：在栈上分配内存块（类似 C 的 `alloca`）。
    *   速度极快，无需 GC。
    *   生命周期仅限当前方法，不可返回给外部。
    *   **Unity 实战**：配合 `Span<T>` 使用，处理临时的小块数据缓冲区。

```csharp
unsafe {
    int* p = stackalloc int[100];
    for (int i = 0; i < 100; i++) {
        p[i] = i; // 飞快的数组操作
    }
}
```
*注意：在 Unity 2018+ 中，使用 C# 7.2 的 `Span<T>` 可以安全地获得类似性能，无需写 unsafe。*

---

## 面试自测题
1.  **GC Roots 有哪些？**（静态变量、栈变量、寄存器、GCHandle）
2.  **Struct 可以继承 Struct 吗？**（不行，struct 是密封的，只能实现接口）
3.  **Dictionary 的 Key 如果是 Struct，需要注意什么？**（必须重写 `GetHashCode` 和 `Equals`，否则默认实现会涉及反射和装箱，性能极差）
4.  **String 为什么要设计成不可变的？**（安全性、哈希码缓存、字符串驻留池优化）
