# Unity Shader 基础概念入门

## 1. 什么是 Shader？

简单来说，**Shader（着色器）** 是一段运行在 GPU（图形处理器）上的小程序，它负责告诉计算机如何绘制物体。

想象你是一个画家（GPU）：
1.  **模型（Mesh）** 提供了物体的形状（线稿）。
2.  **Shader** 决定了物体的颜色、光照、纹理等表面属性（上色规则）。

在 Unity 中，我们通常使用 **ShaderLab** 语言来编写 Shader，它包裹了实际的着色器代码（通常是 HLSL/CG）。

---

## 2. Unity Shader 的基本结构

一个标准的 Unity Shader 文件结构如下：

```shader
Shader "MyShader/SimpleShader" { // 1. Shader 名字，会在材质面板中显示
    Properties {
        // 2. 属性定义：在 Inspector 面板中暴露的变量（如颜色、贴图）
        _MainColor ("Base Color", Color) = (1,1,1,1)
    }
    
    SubShader {
        // 3. 子着色器：针对不同显卡性能的方案
        // Unity 会从上到下寻找第一个当前显卡支持的 SubShader 执行
        
        Tags { "RenderType"="Opaque" } // 标签：告诉 Unity 如何以及何时渲染这个对象
        
        Pass {
            // 4. Pass：一次完整的渲染流程
            
            CGPROGRAM // 开始编写 HLSL 代码
            
            #pragma vertex vert // 定义顶点着色器函数名为 vert
            #pragma fragment frag // 定义片元着色器函数名为 frag
            
            #include "UnityCG.cginc" // 引入 Unity 内置辅助函数库
            
            // ... 具体的 Shader 代码 ...
            
            ENDCG // 结束 HLSL 代码
        }
    }
    
    FallBack "Diffuse" // 5. 后备方案：如果上面的 SubShader 都不支持，就用这个内置 Shader
}
```

### 关键部分详解

#### Properties (属性)
这里定义的变量会显示在材质球（Material）的 Inspector 面板上，方便美术调整。
常见格式：`_Name ("Display Name", Type) = DefaultValue`

| 类型 | 描述 | 例子 |
| :--- | :--- | :--- |
| **Color** | 颜色 (RGBA) | `_Color ("Color", Color) = (1,1,1,1)` |
| **Vector** | 向量 (XYZW) | `_Vector ("Vector", Vector) = (0,0,0,0)` |
| **Float** / **Range** | 浮点数 / 范围 | `_Gloss ("Gloss", Range(0,1)) = 0.5` |
| **2D** | 2D 纹理 | `_MainTex ("Texture", 2D) = "white" {}` |

#### SubShader (子着色器)
一个 Shader 文件可以包含多个 SubShader。Unity 会根据硬件能力自动选择最合适的一个。

#### Pass (通道)
一个 SubShader 可以包含一个或多个 Pass。每个 Pass 都会让物体被渲染一次。大部分简单的 Shader 只需要一个 Pass。

---

## 3. 渲染流水线简述 (The Rendering Pipeline)

要理解 Shader，必须理解数据是如何流动的。最核心的两个阶段是：

### 3.1 顶点着色器 (Vertex Shader)
*   **输入**：模型顶点的原始信息（位置、法线、UV坐标等）。
*   **任务**：
    1.  **坐标变换**：把顶点从 **模型空间** (Local Space) 转换到 **裁剪空间** (Clip Space)。简单说就是把 3D 世界里的点换算到 2D 屏幕上的位置。
    2.  **准备数据**：计算一些后续需要的数据（如世界空间法线、光照方向）传给片元着色器。
*   **输出**：裁剪空间下的顶点位置 + 自定义数据。
*   **执行次数**：每个顶点执行一次。

### 3.2 片元着色器 (Fragment Shader / Pixel Shader)
*   **输入**：从顶点着色器传过来的数据（经过插值）。
*   **任务**：计算屏幕上每个像素的最终颜色。
    *   读取纹理（采样）。
    *   计算光照（漫反射、高光）。
    *   计算阴影、透明度等。
*   **输出**：该像素的颜色值 (RGBA)。
*   **执行次数**：屏幕上该物体覆盖的每个像素执行一次。

**数据流向：**
`模型数据 (Mesh)` -> `Vertex Shader` -> `(光栅化插值)` -> `Fragment Shader` -> `屏幕显示`

---

## 4. 你的第一个 Shader 任务

在接下来的文档中，我们将动手写代码。你需要准备：
1.  Unity 编辑器。
2.  创建一个新的 Material（右键 -> Create -> Material）。
3.  创建一个新的 Shader（右键 -> Create -> Shader -> Unlit Shader），命名为 `01_MyFirstShader`。
4.  双击打开 Shader 文件，清空内容，准备跟随教程编写。

下一章：我们将编写一个最简单的“纯色”Shader，并让它动起来。
