# 02. C# 高级特性与异步编程实战

## 1. 委托与事件：从原理到工程实践

### 1.1 委托 (Delegate) 的本质
*   **IL 视角**：委托编译后是一个类，继承自 `System.MulticastDelegate`。
*   **内存布局**：
    *   `_target`：方法所属的对象实例（对于静态方法为 null）。
    *   `_methodPtr`：方法的内存地址。
    *   `_invocationList`：如果是多播委托，这里存储一个委托数组。
*   **多播委托 (MulticastDelegate)**：可以通过 `+=` 链接多个方法。调用时按添加顺序依次执行。
    *   **坑点**：如果其中一个方法抛出异常，后续方法将不会执行！且调用方只能捕获最后一个异常。
    *   **解决**：使用 `GetInvocationList()` 手动遍历调用，包裹 `try-catch`。

### 1.2 事件 (Event) vs 委托
*   **事件是对委托的封装**：就像属性 (`Property`) 是对字段 (`Field`) 的封装一样。
*   **安全性**：
    *   **委托**：外部类可以赋值 (`= null`)，甚至直接调用 (`Invoke()`)，破坏封装性。
    *   **事件**：外部类只能 `+=` (订阅) 和 `-=` (取消订阅)，不能赋值或直接触发。触发权在定义事件的类内部。
*   **最佳实践**：总是声明 `event Action` 或 `event EventHandler`，除非你需要作为参数传递回调。

### 1.3 闭包 (Closure) 陷阱
当 Lambda 表达式引用了外部变量时，编译器会生成一个**匿名类**来持有这些变量。

```csharp
void Start() {
    int id = 10;
    Action action = () => Debug.Log(id); // id 被捕获
    // 编译器生成类似：
    // class DisplayClass { public int id; void Method() { Log(id); } }
    // action = new DisplayClass { id = 10 }.Method;
}
```

*   **陷阱：循环变量捕获**
```csharp
List<Action> actions = new List<Action>();
for (int i = 0; i < 5; i++) {
    actions.Add(() => Debug.Log(i)); // 错误！i 是共享变量
}
// 执行结果：5, 5, 5, 5, 5
// 原因：匿名类只生成了一个实例，i 是其中的字段，循环结束后 i=5。

// 修正：
for (int i = 0; i < 5; i++) {
    int temp = i; // 每次循环声明新变量
    actions.Add(() => Debug.Log(temp)); // 正确捕获 temp
}
```

---

## 2. 泛型 (Generics) 与协变逆变

### 2.1 泛型的实现机制
*   **代码膨胀 (Code Bloat)**：
    *   **引用类型**：`List<string>` 和 `List<object>` 在底层共享同一份机器码（因为指针大小相同）。
    *   **值类型**：`List<int>` 和 `List<float>` 会生成两份完全不同的机器码（因为数据大小和操作指令不同）。
*   **静态构造函数**：每个封闭泛型类型（如 `MyClass<int>` 和 `MyClass<string>`）都会触发一次静态构造函数。

### 2.2 约束 (Constraints)
使用 `where` 关键字限制泛型类型，避免运行时类型检查和装箱。
*   `where T : struct` (值类型)
*   `where T : class` (引用类型)
*   `where T : new()` (必须有无参构造)
*   `where T : Component` (必须是 Unity 组件)

### 2.3 协变 (Covariance) 与 逆变 (Contravariance)
*   **协变 (`out`)**：`IEnumerable<out T>`。如果 `Cat` 是 `Animal`，则 `IEnumerable<Cat>` 是 `IEnumerable<Animal>`。
    *   只能作为返回值输出，不能作为参数输入。
*   **逆变 (`in`)**：`Action<in T>`。如果 `Cat` 是 `Animal`，则 `Action<Animal>` 可以赋值给 `Action<Cat>`（能处理所有动物的方法当然能处理猫）。
    *   只能作为参数输入，不能作为返回值输出。
*   **记忆口诀**：**“出协入逆”** (Out Covariant, In Contravariant)。

---

## 3. 异步编程：Task, Async/Await 与 Unity

### 3.1 为什么需要异步？
Unity 主线程（Main Thread）必须以 60fps+ 运行。任何超过 16ms 的操作（如 I/O、复杂计算、网络请求）如果同步执行，都会导致画面卡顿。

### 3.2 协程 (Coroutine) vs Async/Await
| 特性 | 协程 (IEnumerator) | Async/Await (Task) |
| :--- | :--- | :--- |
| **本质** | 迭代器模式，分帧执行 | 状态机模式，基于 Task 调度 |
| **线程** | **始终在主线程** | 可在**后台线程**运行 (Task.Run) |
| **返回值** | 无 (只能 yield return) | 可返回 `Task<T>` |
| **异常处理** | 难以捕获 (try-catch 无效) | 完美支持 try-catch |
| **组合性** | 较弱 (StartCoroutine 嵌套) | 极强 (WhenAll, WhenAny) |

### 3.3 UniTask：Unity 的最佳拍档
原生 `Task` 有两个问题：
1.  **GC Alloc**：`Task` 对象本身是类，分配有开销。
2.  **线程切换**：`await` 默认会尝试回到原始上下文，但在 Unity 中处理不当容易死锁或跨线程访问组件报错。

**UniTask (库)** 解决了这些问题：
*   **基于 struct**：零分配 (Zero Allocation)。
*   **主线程敏感**：默认 `await` 回到 Unity 主线程，可安全操作 GameObject。
*   **功能丰富**：支持 `WaitUntil`, `DelayFrame`, `NextFrame` 等 Unity 常用功能。

```csharp
// 推荐写法 (使用 UniTask)
public async UniTaskVoid LoadAssetAsync() {
    try {
        // 切换到线程池做繁重计算
        await UniTask.SwitchToThreadPool();
        var data = CalculateHugeData();
        
        // 切回主线程应用数据
        await UniTask.SwitchToMainThread();
        transform.position = data.position;
    }
    catch (Exception e) {
        Debug.LogError(e);
    }
}
```

### 3.4 异步死锁 (Deadlock)
如果在主线程直接调用 `.Result` 或 `.Wait()` 等待一个需要切回主线程的 Task，会导致死锁。
*   **原因**：主线程被阻塞在 Wait，而 Task 完成后想切回主线程执行后续代码（Continuation），却发现主线程被占用了。
*   **解决**：**永远不要在同步方法中阻塞等待异步任务**。使用 `async void` (仅限顶层事件) 或一直 `await` 到底。

---

## 4. 反射 (Reflection) 与 Attribute

### 4.1 反射的性能开销
*   **元数据查找**：字符串匹配，慢。
*   **动态调用**：`Invoke` 无法内联，且参数需要装箱为 `object[]`。
*   **优化**：
    1.  **缓存**：只查找一次 `MethodInfo`，存入 `Dictionary`。
    2.  **Delegate.CreateDelegate**：将反射方法转换为强类型委托，性能接近原生调用。

### 4.2 Attribute (特性)
*   **元数据标签**：本身不执行逻辑，只是挂在类/方法上的数据。
*   **获取**：必须通过反射 `GetCustomAttributes` 获取。
*   **Unity 实战**：
    *   `[SerializeField]`：序列化私有字段。
    *   `[ContextMenu]`：在 Inspector 右键菜单添加按钮。
    *   `[Range(0, 1)]`：限制数值范围。

---

## 面试自测题
1.  **Delegate 和 Event 的区别是什么？**（Event 是对 Delegate 的封装，限制了外部赋值和调用权限，符合观察者模式）
2.  **协程是多线程吗？**（不是，协程是主线程的分帧执行，本质是迭代器）
3.  **Task.Run 和 Task.Factory.StartNew 有什么区别？**（Task.Run 是简化版，默认使用线程池，更安全；StartNew 参数更多，但默认调度器不同，容易踩坑）
4.  **泛型 T 为什么不能直接 `new T()`？**（因为编译器不知道 T是否有无参构造，必须加 `where T : new()` 约束）
