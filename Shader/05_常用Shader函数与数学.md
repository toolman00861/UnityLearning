# 05. 常用 Shader 函数与数学：造就万物的魔法

Shader 的本质是数学。掌握几个核心函数，就能做出无数种效果。

## 1. 基础运算

| 函数 | 描述 | 用途 |
| :--- | :--- | :--- |
| **saturate(x)** | 把 x 截取到 [0, 1] 范围 | 防止颜色溢出（变负数或太亮） |
| **clamp(x, min, max)** | 把 x 截取到 [min, max] 范围 | 限制数值范围 |
| **lerp(a, b, t)** | 线性插值：(1-t)*a + t*b | 混合两个颜色、纹理、数值 |
| **abs(x)** | 取绝对值 | 制作对称效果 |
| **frac(x)** | 取小数部分 (x - floor(x)) | 制作重复图案（如条纹、格子） |

## 2. 形状控制

| 函数 | 描述 | 用途 |
| :--- | :--- | :--- |
| **step(edge, x)** | 阶跃函数：如果 x >= edge 返回 1，否则返回 0 | 制作硬边缘、开关、遮罩 |
| **smoothstep(min, max, x)** | 平滑阶跃：在 min 和 max 之间平滑过渡 (0->1) | 制作软边缘、渐变 |

## 3. 几何运算

| 函数 | 描述 | 用途 |
| :--- | :--- | :--- |
| **length(v)** | 计算向量长度 | 计算距离 |
| **distance(p1, p2)** | 计算两点距离 | 制作圆形、光晕 |
| **dot(a, b)** | 点积 | 计算光照、判断方向是否一致 |
| **cross(a, b)** | 叉积 | 计算垂直向量（如求法线） |
| **reflect(i, n)** | 计算反射向量 | 制作镜面反射效果 |

---

## 实战：用数学画一个圆

我们不使用贴图，只用数学函数在 Quad 上画一个圆。

创建一个新 Shader `05_MathCircle`：

```shader
Shader "MyShader/05_MathCircle" {
    Properties {
        _CircleColor ("Color", Color) = (1, 0, 0, 1)
        _Radius ("Radius", Range(0, 0.5)) = 0.3
        _Smooth ("Smoothness", Range(0, 0.1)) = 0.01
    }
    SubShader {
        Tags { "RenderType"="Transparent" "Queue"="Transparent" }
        Blend SrcAlpha OneMinusSrcAlpha // 开启混合模式以显示圆形边缘的透明度
        
        Pass {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #include "UnityCG.cginc"

            struct appdata {
                float4 vertex : POSITION;
                float2 uv : TEXCOORD0;
            };

            struct v2f {
                float4 pos : SV_POSITION;
                float2 uv : TEXCOORD0;
            };

            fixed4 _CircleColor;
            float _Radius;
            float _Smooth;

            v2f vert (appdata v) {
                v2f o;
                o.pos = UnityObjectToClipPos(v.vertex);
                o.uv = v.uv; // 传递原始 UV (0~1)
                return o;
            }

            fixed4 frag (v2f i) : SV_Target {
                // 1. 把 UV 坐标中心移到 (0.5, 0.5) -> (-0.5, 0.5)
                float2 centerUV = i.uv - 0.5;
                
                // 2. 计算当前像素到中心的距离
                float dist = length(centerUV);
                
                // 3. 使用 smoothstep 计算圆形遮罩
                // 如果距离小于半径，返回 1 (显示颜色)
                // 如果距离大于半径，返回 0 (透明)
                // 使用 smoothstep 在边缘产生平滑过渡（抗锯齿）
                float mask = smoothstep(_Radius + _Smooth, _Radius, dist);
                
                // 4. 输出颜色
                // 遮罩值为 0 时透明，为 1 时显示颜色
                return fixed4(_CircleColor.rgb, mask * _CircleColor.a);
            }
            ENDCG
        }
    }
}
```

## 关键代码解析

1.  `i.uv - 0.5`：
    默认 UV 是左下角 (0,0) 到右上角 (1,1)。减去 0.5 后，中心点变为 (0,0)，范围变为 -0.5 到 0.5。这样方便计算距离。

2.  `length(centerUV)`：
    计算当前像素点到中心点 (0,0) 的距离。

3.  `smoothstep(_Radius + _Smooth, _Radius, dist)`：
    *   当 `dist` < `_Radius` 时，结果为 1（圆内）。
    *   当 `dist` > `_Radius + _Smooth` 时，结果为 0（圆外）。
    *   中间区域平滑过渡（边缘虚化）。
    *   **注意**：这里 min 和 max 是反过来的（大值在前，小值在后），是为了让圆内是 1，圆外是 0。

## 练习
1.  修改 `_Radius` 和 `_Smooth` 观察效果。
2.  尝试画一个**环形**。
    *   提示：计算两个圆的遮罩（大圆减小圆），或者使用 `abs(dist - _Radius)`。
3.  尝试让圆动起来。
    *   `float2 centerUV = i.uv - 0.5 + float2(sin(_Time.y)*0.2, 0);` 让圆左右移动。

---

## 总结
恭喜你！你已经掌握了 Shader 的核心基础：
1.  **结构**：Properties, SubShader, Pass。
2.  **流水线**：Vertex (顶点变换) -> Fragment (像素着色)。
3.  **光照**：Dot Product, Lambert, Blinn-Phong。
4.  **纹理**：UV 坐标, tex2D 采样。
5.  **透明**：Alpha Test, Alpha Blend, ZWrite。
6.  **数学**：利用函数程序化生成图形。

接下来，你可以尝试去 ShaderToy 网站看看大神们的作品，或者深入学习 PBR (物理渲染) 和后处理特效。
