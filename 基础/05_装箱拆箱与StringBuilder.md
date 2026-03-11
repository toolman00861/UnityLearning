# C# 核心机制：装箱、拆箱与 StringBuilder

## 1. 装箱 (Boxing) 与 拆箱 (Unboxing)

### 1.1 基本概念
*   **装箱 (Boxing)**: 将**值类型** (Value Type) 转换为 `object` 类型或由此值类型实现的任何**接口类型**的过程。
    *   *底层动作*: CLR 会在**堆 (Heap)** 上分配一块新内存，将**栈 (Stack)** 上值类型的数据复制到这块堆内存中，并返回这块内存的地址（引用）。
*   **拆箱 (Unboxing)**: 将 `object` 类型或接口类型转换为**值类型**的过程。
    *   *底层动作*: CLR 检查对象实例是否是给定值类型的装箱值（类型安全检查），然后将值从堆复制回栈。

### 1.2 代码表现与内存图解

```csharp
int i = 123;      // 值类型，数据直接存在栈上
object o = i;     // 【装箱】：在堆上申请内存 -> 复制 123 -> o 指向堆地址
int j = (int)o;   // 【拆箱】：检查 o 是否是 int -> 复制堆上的 123 到栈上的 j

// 注意：拆箱必须转换为原本的类型，否则抛出 InvalidCastException
long x = (long)o; // 错误！即使 int 可以隐式转为 long，但拆箱必须精确匹配
long y = (long)(int)o; // 正确：先拆箱为 int，再隐式转换为 long
```

### 1.3 隐式装箱陷阱 (Unity 面试高频)

在日常开发中，显式的 `object o = i` 很容易识别，但隐式装箱往往是性能杀手：

1.  **`String.Format` / `Debug.Log` / `string + object`**:
    ```csharp
    int score = 100;
    // Debug.Log 的参数签名是 (object message)
    Debug.Log("Score: " + score); 
    // 过程：score (int) -> 装箱为 object -> 调用 object.ToString()
    
    // 【优化】：主动调用 .ToString()
    // score.ToString() 返回 string (引用类型)，直接传递给 Log，避免了装箱
    Debug.Log("Score: " + score.ToString()); 
    ```

2.  **非泛型集合 (ArrayList)**:
    ```csharp
    // 极不推荐使用
    ArrayList list = new ArrayList(); 
    list.Add(10); // 10 被装箱为 object 存入
    int val = (int)list[0]; // 取出时拆箱
    
    // 【优化】：使用泛型 List<T>
    List<int> listGeneric = new List<int>(); 
    listGeneric.Add(10); // 无装箱，直接在底层数组存 int
    ```

3.  **使用接口引用值类型**:
    ```csharp
    interface IReset { void Reset(); }
    struct PlayerState : IReset { 
        public void Reset() { } 
    }
    
    PlayerState state = new PlayerState();
    IReset resetable = state; // 【装箱】！接口是引用类型
    resetable.Reset(); 
    ```

### 1.4 性能影响
*   **GC 压力 (Garbage Collection)**: 装箱会在堆上分配内存，产生临时对象。这些对象用完即弃，会迅速填满 0 代堆 (Generation 0)，导致 GC 频繁触发，引起游戏掉帧 (Spike)。
*   **CPU 消耗**: 内存分配（寻找空闲内存块）和数据拷贝都需要 CPU 周期。

---

## 2. StringBuilder 详解

### 2.1 为什么需要它？ (String 的不可变性)
C# 中的 `string` 是**不可变 (Immutable)** 的。
```csharp
string s = "Hello";
s += " World"; 
```
执行 `+=` 时，并不会修改原有的 "Hello" 内存，而是：
1.  申请一块新内存，大小为 "Hello" + " World" 的长度。
2.  复制 "Hello"。
3.  复制 " World"。
4.  `s` 指向新内存。旧的 "Hello" 变成垃圾等待 GC。

如果是循环拼接 1000 次，就会产生 1000 个中间字符串垃圾。

### 2.2 StringBuilder 原理
`System.Text.StringBuilder` 内部维护一个**可变的字符数组 (`char[]`)**。
*   `Append` 时，直接在数组空闲位置写入字符。
*   只有当内部数组容量不足时，才会创建一个更大的新数组（通常是 2 倍），并将旧数据复制过去。
*   `ToString()` 时，才会真正生成一个 `string` 对象。

### 2.3 常用 API 与 代码示例

```csharp
using System.Text;

public class StringBuilderDemo : MonoBehaviour
{
    // 【最佳实践】：作为成员变量复用，避免每次 Update 都 new
    private StringBuilder _sb = new StringBuilder(512); // 预设容量 512，减少扩容

    void Update()
    {
        // 1. 清空内容（不释放内部数组内存，只重置计数器）
        _sb.Clear();

        // 2. 追加
        _sb.Append("Frame: ");
        _sb.Append(Time.frameCount); // 内部处理了 int，避免装箱 (取决于 .NET 版本，通常有重载)
        
        // 3. 追加行 (自动加 \n)
        _sb.AppendLine(); 
        
        // 4. 格式化追加 (替代 String.Format)
        _sb.AppendFormat("Pos: ({0:F1}, {1:F1})", transform.position.x, transform.position.y);

        // 5. 插入与替换
        _sb.Insert(0, "[Log] ");
        _sb.Replace("Frame", "F");

        // 6. 输出
        string result = _sb.ToString();
        // Debug.Log(result);
    }
}
```

### 2.4 性能对比与使用场景
| 操作 | 推荐方式 | 原因 |
| :--- | :--- | :--- |
| **少量拼接** (2-3 个) | `string a = b + c;` | 编译器会自动优化为 `String.Concat`，开销极小，代码可读性好。 |
| **循环拼接** | `StringBuilder` | 避免产生大量中间垃圾对象。 |
| **大量拼接** | `StringBuilder` | 性能提升在数量级以上。 |
| **字符串修改** | `StringBuilder` | String 不支持修改，只能替换生成新的。 |

### 2.5 内存优化技巧
1.  **预设容量 (Capacity)**: `new StringBuilder(1024)`。如果你知道大概字符串有多长，初始化时指定容量，可以避免中间的数组扩容（扩容需要申请新数组+拷贝）。
2.  **重用实例**: 不要每次函数调用都 `new StringBuilder()`。在类中定义一个 `private` 字段，配合 `Clear()` 反复使用。
