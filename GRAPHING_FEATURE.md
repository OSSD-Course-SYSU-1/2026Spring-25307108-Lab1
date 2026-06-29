# 卡片计算器绘图功能 — 技术描述文档

## 1. 功能概述

本次迭代在计算器应用中集成了完整的**函数曲线绘图**模块，使应用从单纯的算术工具升级为**轻量级数学工作台**。用户可通过专用的表达式输入面板编写含变量 `x` 的数学函数（支持 12 种初等函数、常数和幂运算），在 Canvas 画布上实时渲染函数曲线，并支持单指拖拽平移与双指捏合缩放的交互式视口操作。曲线支持叠加绘制，以不同颜色区分多条函数。

| 维度 | 绘图前 | 绘图后 |
|------|--------|--------|
| 应用定位 | 计算器 | 计算器 + 函数可视化工具 |
| 表达式引擎 | 仅算术（四则 + 百分比） | 算术 + 函数（12 种）+ 变量 |
| UI 页面 | 2 页（主页面 / 历史） | 3 页（主页面 / 历史 / 绘图） |
| 交互方式 | 按键输入 | 按键输入 + 手势拖拽 + 手势缩放 |
| 显示能力 | 文本表达式 | 文本 + Canvas 光栅图形 |

## 2. 系统架构

### 2.1 页面导航流

```
主计算器页面
├── [🖌️] → 绘图页面 (showGraph = true)
│   ├── 顶栏：返回 / 标题 / 缩放控制
│   ├── 表达式预览栏
│   ├── Canvas 画布（手势交互）
│   └── 输入面板（可折叠）
└── [📚]  → 历史记录页面 (showHistory = true)
```

### 2.2 新增状态模型

```typescript
@State showGraph: boolean = false;          // 绘图页面可见性
@State curves: CurveItem[] = [];            // 已绘制曲线集合
@State currentColor: string = '#2563EB';    // 当前绘制颜色
@State graphExpr: string = '';              // 正在编辑的表达式
@State showGraphControls: boolean = true;   // 输入面板展开/折叠
@State xMin: number = -10;  @State xMax: number = 10;  // X 轴视口范围
@State yMin: number = -10;  @State yMax: number = 10;  // Y 轴视口范围

interface CurveItem {
  expression: string;   // 原始函数表达式
  color: string;        // 曲线颜色
}
```

## 3. 函数表达式引擎

### 3.1 支持的数学元素

| 类别 | 元素 | 输入标识 | 说明 |
|------|------|----------|------|
| 三角函数 | sin, cos, tan, cot, sec, csc | `sin(`, `cos(` 等 | 输入自动补齐左括号 |
| 反双曲 | — | — | 可通过 `1/tan(x)` 等组合实现 |
| 根号 | sqrt, cbrt | `sqrt(`, `cbrt(` | 平方根 / 立方根 |
| 对数 | ln, log10 | `ln(`, `log10(` | 自然对数 / 常用对数 |
| 指数 | exp, ^ | `exp(`, `^` | eˣ / 通用幂运算 |
| 绝对值 | abs | `abs(` | \|x\| |
| 常数 | pi, e | `pi`, `e` | π ≈ 3.14159 / e ≈ 2.71828 |
| 变量 | x | `x` | 自变量 |

### 3.2 表达式解析流水线

```
原始输入字符串 "sin(x)+x^2"
   │
   ▼
functionTokenize()        词法分析
   │  输入: "sin(x)+x^2"
   │  输出: ['sin', '(', 'x', ')', '+', 'x', '^', '2']
   │
   ▼
functionPostfix()         语法分析（中缀→后缀）
   │  输出: ['x', 'sin', 'x', '2', '^', '+']
   │
   ▼
evaluateFunctionPostfix() 后缀求值（代入 x 数值）
   │  输出: 数值结果
   ▼
Canvas 绘制像素点
```

### 3.3 词法分析 `functionTokenize()`

逐字符扫描输入字符串，识别以下 token 类型：

| Token 类型 | 示例 | 处理规则 |
|-----------|------|----------|
| 数字（含小数） | `3.14` | 连续数字字符合并 |
| 变量 | `x` | 独立 token |
| 运算符 | `+ - * / ^` | 独立 token |
| 一元负号 | `-x` | 位于表达式首 / `(` 后 / `,` 后 / 运算符后时识别为 `u-` |
| 括号 | `( )` | 独立 token |
| 逗号 | `,` | 独立 token（函数参数分隔） |
| 函数名 | `sin`, `cos`, ... | 匹配 FUNCTIONS 预定义列表 |
| 常数 | `pi`, `e` | 即时替换为数值 |
| 空白 | ` ` | 跳过 |

**一元负号检测逻辑**：

```typescript
if (ch === '-' && (tokens.length === 0 ||
    tokens[tokens.length-1] === '(' ||
    tokens[tokens.length-1] === ',' ||
    isOperator(tokens[tokens.length-1]))) {
  tokens.push('u-');  // 标记为「一元减」区别于二元减法
}
```

这是表达式解析中最易出错的环节。将 `-x²` 与 `x-y` 中的 `-` 正确区分为一元/二元操作符，保证了数学语义的准确。

### 3.4 语法分析 `functionPostfix()`

将中缀 token 序列转为后缀（逆波兰）表示，处理优先级规则：

| 优先级 | 元素 | 规则 |
|--------|------|------|
| 最高 | 函数调用 `sin(`, `ln(` ... | 遇到 `)` 时弹出函数标记 |
| 高 | 括号 `()` | 左括号入栈，右括号弹至匹配 |
| 中高 | 幂 `^` | 右结合（特殊处理） |
| 中 | 乘除 `* /` | 左结合 |
| 低 | 加减 `+ -` | 左结合 |
| 最低 | 逗号 `,` | 弹至左括号 |

函数调用在栈中作为标记：`sin(expr)` 被解析为 `expr sin`（后缀中的函数应用位于参数之后）。

### 3.5 后缀求值 `evaluateFunctionPostfix()`

遍历后缀表达式，代入具体 `x` 值计算：

```typescript
function evaluateFunctionPostfix(postfix: string[], x: number): number {
  const stack: number[] = [];
  for (const token of postfix) {
    if (!isNaN(Number(token)))  → stack.push(Number(token));
    else if (token === 'x')     → stack.push(x);
    else if (token === 'u-')    → stack.push(-stack.pop()!);
    else if (isFunction(token)) → stack.push(applyFunction(token, stack.pop()!));
    else if (isOperator(token)) → 二元运算
  }
  return stack[0] ?? 0;
}
```

**12 种函数的计算实现**：

| 函数 | 实现 | 数学含义 |
|------|------|----------|
| `sin` | `Math.sin(a)` | 正弦 |
| `cos` | `Math.cos(a)` | 余弦 |
| `tan` | `Math.tan(a)` | 正切 |
| `cot` | `1 / Math.tan(a)` | 余切 |
| `sec` | `1 / Math.cos(a)` | 正割 |
| `csc` | `1 / Math.sin(a)` | 余割 |
| `abs` | `Math.abs(a)` | 绝对值 |
| `sqrt` | `Math.sqrt(a)` | 平方根 |
| `cbrt` | `Math.cbrt(a)` | 立方根 |
| `ln` | `Math.log(a)` | 自然对数 |
| `log10` | `Math.log10(a)` | 常用对数 |
| `exp` | `Math.exp(a)` | e 的幂 |

## 4. Canvas 绘图引擎

### 4.1 坐标映射

```
数学坐标系 (xMin..xMax, yMin..yMax)
          │
          │  mapX / mapY
          ▼
Canvas 像素坐标系 (0..width, 0..height)

mapX(val) = (val - xMin) / (xMax - xMin) × width
mapY(val) = height - (val - yMin) / (yMax - yMin) × height
                                    ↑ Y 轴翻转（屏幕 Y 轴向下）
```

### 4.2 绘制流程 `drawGraph()`

```
1. clearRect(0, 0, w, h)              ← 清空画布
2. 绘制坐标轴（x=0 线 + y=0 线）       ← 白色实线，1px
3. 绘制刻度线和数值标签                ← 自适应步长
   ├─ X 轴刻度：stepX 间隔，标记 v 值
   └─ Y 轴刻度：stepY 间隔，标记 v 值
4. 遍历 curves[] 绘制曲线               ← 每条曲线采样 500 点
   ├─ 剥离 "y=" 前缀
   ├─ 以 step = (xMax-xMin)/500 递增
   ├─ functionTokenize → functionPostfix → evaluateFunctionPostfix
   ├─ NaN/Infinity 跳过（函数间断点自然断开）
   └─ moveTo / lineTo 连接有效点
```

### 4.3 刻度步长自适应算法

```typescript
stepForRange(range: number): number {
  let rough = range / 5;                          // 目标：约 5 条刻度线
  let pow10 = Math.pow(10, Math.floor(Math.log10(rough))); // 数量级
  let normalized = rough / pow10;                 // 归一化到 [1,10)
  if (normalized < 1.5) return pow10;             // → 1, 10, 100...
  if (normalized < 3)   return 2 * pow10;         // → 2, 20, 200...
  if (normalized < 7)   return 5 * pow10;         // → 5, 50, 500...
  return 10 * pow10;                              // → 10, 100, 1000...
}
```

| 视口范围 | rough | pow10 | normalized | 步长 |
|----------|-------|-------|------------|------|
| 0~10 | 2 | 1 | 2 | 2 |
| 0~5 | 1 | 1 | 1 | 1 |
| 0~100 | 20 | 10 | 2 | 20 |
| 0~3 | 0.6 | 0.1 | 6 | 0.5 |
| -1~1 | 0.4 | 0.1 | 4 | 0.5 |

算法保证刻度值始终为 1/2/5 × 10ⁿ 的「好」数字，避免出现 0.7 / 1.3 之类的奇怪刻度。

### 4.4 曲线采样策略

默认在视口 X 范围内**均匀采样 500 个点**，以 `moveTo` / `lineTo` 方式连接。对于典型 360dp 宽的卡片 Canvas，500 点意味着约 1.4 像素/点，保证曲线视觉平滑且无冗余计算。

**间断点处理**：当某采样点的函数值返回 `NaN` 或 `±Infinity`（如 `tan(π/2)`），绘制跳过该点并将 `first` 标志重置为 `true`，下一点重新 `moveTo`——曲线在间断处自然断开而非绘制伪影跨接。

## 5. 交互式视口操作

### 5.1 手势系统

```typescript
GestureGroup(GestureMode.Parallel,
  PanGesture({ fingers: 1, distance: 5 })   // 单指拖拽平移
    .onActionUpdate((event) => {
      // 将像素位移映射为坐标系位移
      xMin -= dx / sensitivity * rangeX;
      xMax -= dx / sensitivity * rangeX;
      yMin += dy / sensitivity * rangeY;
      yMax += dy / sensitivity * rangeY;
      drawGraph();  // 实时重绘
    }),
  PinchGesture({ fingers: 2, distance: 5 }) // 双指捏合缩放
    .onActionUpdate((event) => {
      // 以视口中心为锚点缩放
      scaleChange = event.scale;
      newRangeX = (xMax - xMin) / scaleChange;
      newRangeY = (yMax - yMin) / scaleChange;
      // 保持中心点不变
      xMin = cx - newRangeX/2;  xMax = cx + newRangeX/2;
      yMin = cy - newRangeY/2;  yMax = cy + newRangeY/2;
      drawGraph();  // 实时重绘
    })
)
```

| 手势 | 手指数 | 最小距离 | 效果 |
|------|--------|----------|------|
| Pan (拖拽) | 1 | 5px | 平移坐标系，灵敏度 200（像素→坐标单位） |
| Pinch (捏合) | 2 | 5px | 以画布中心为锚点等比缩放 |

两种手势以 `GestureMode.Parallel` 并行注册，同时支持——用户可以一边拖动一边缩放，与 Google Maps 的交互模型一致。

### 5.2 按钮缩放

顶栏提供 `+` / `−` 按钮作为手势缩放的补充：

| 按钮 | 操作 | 效果 |
|------|------|------|
| `+` | `zoomGraph(0.5)` | 放大（范围减半） |
| `−` | `zoomGraph(2.0)` | 缩小（范围加倍） |

也以视口中心为锚点，与手势缩放行为一致。

## 6. 绘图输入面板设计

### 6.1 面板布局 (8 行 × 4 列)

```
Row 1: [y]  [=]  [x]  [⌫]     ← 变量快捷输入（前三键橙金色渐变高亮）
Row 2: [sin][cos][tan][cot]    ← 三角函数
Row 3: [ln] [sqrt][^] [abs]    ← 对数/根号/幂/绝对值
Row 4: [7]  [8]  [9]  [/]     ← 数字与运算符
Row 5: [4]  [5]  [6]  [*]
Row 6: [1]  [2]  [3]  [-]
Row 7: [0]  [.]  [(]  [)]     ← 数字 + 括号
Row 8: [pi] [e]  [00] [+]     ← 常数 + 数字辅助
─────────────────────────────────
     [  绘制  ] [  清空  ]      ← 操作按钮行
```

### 6.2 按键行为

| 按键 | 行为 |
|------|------|
| `y` `=` `x` | 追加字符串（橙金色高亮） |
| `sin` `cos` ... | 追加函数名 + `(`（自动补左括号） |
| `⌫` | 删除末尾字符 |
| 数字/运算符/括号 | 标准追加 |
| `pi` `e` | 追加常数名（解析时替换为数值） |
| **绘制** | 解析表达式 → 添加曲线 → 重绘 → 折叠面板 |
| **清空** | 清空当前输入 |

### 6.3 表达式预览栏

面板上方显示实时输入的表达式：

```
┌──────────────────────────────┐
│  y=sin(x)+x^2               │  ← 16fp 白色，最多 2 行右对齐
└──────────────────────────────┘
```

背景半透明深灰 + 毛玻璃效果，与整体暗黑主题一致。

### 6.4 面板折叠策略

按下「绘制」后 `showGraphControls` 自动置为 `false`，面板收起，Canvas 扩展至全屏——用户可以无障碍地拖拽/缩放观察曲线全貌。顶栏出现 `✎` 按钮，点击可重新展开面板添加第二条曲线。

## 7. 多曲线叠加

### 7.1 曲线管理

```typescript
interface CurveItem {
  expression: string;    // 如 "sin(x)"
  color: string;         // 如 "#2563EB"
}
```

每条曲线以独立颜色绘制在 Canvas 上。当前版本使用单一颜色 `#2563EB`（蓝色），架构上支持扩展为自动色轮分配。

### 7.2 绘制示例

```
输入: y = sin(x)      ─── 蓝色曲线
输入: y = cos(x)      ─── 蓝色曲线（叠加于同一画布）
输入: y = x^2         ─── 蓝色曲线
```

三条曲线同时可见，坐标轴与刻度线在底层共享。

## 8. 计算器引擎扩展：幂运算符

为支持绘图表达式中的 `x^2`、`2^x` 等写法，计算器核心引擎同步扩展了幂运算支持：

| 变更 | 位置 | 说明 |
|------|------|------|
| `isOperator()` 添加 `'^'` | Line 10 | 识别 `^` 为运算符 |
| `OPERATORLEVELS['^'] = 2` | Line 19 | 优先级高于乘除（1） |
| 优先级重写 | Line 14-17 | `^` 的右结合性判断 |
| `OPERATORHANDLERS['^']` | Line 27 | `Math.pow(a, b)` |
| `getFloatNum('^')` | Line 39 | 精度策略同加减 |

**右结合性处理**：

```typescript
function isPrioritized(first: string, second: string): boolean {
  if (first === '^' && second !== '^') return true;   // ^ 右结合
  return OPERATORLEVELS[first] > OPERATORLEVELS[second];
}
```

幂运算是右结合的：`2^3^2 = 2^(3^2) = 2^9 = 512`，而非 `(2^3)^2 = 64`。此处理保证了数学语义正确。

## 9. UI 组件层级树（绘图模块）

```
showGraph = true
└── Column (全屏容器, #000000)
    ├── Row (顶栏)
    │   ├── Button [←]                    ← 返回主页面
    │   ├── Text '函数绘图'                ← 标题
    │   └── Row (控制按钮组)
    │       ├── Button [🗑]                ← 清空所有曲线
    │       ├── Button [+]                ← 放大
    │       ├── Button [−]                ← 缩小
    │       └── Button [✎] (条件可见)     ← 展开输入面板
    ├── Row (表达式预览)
    │   └── Text(graphExpr)               ← 当前编辑的表达式
    ├── Stack → Canvas (graphCtx)          ← 绘图画布 (layoutWeight:1)
    │   └── GestureGroup
    │       ├── PanGesture (1 指平移)
    │       └── PinchGesture (2 指缩放)
    └── Column (输入面板, 条件可见)
        ├── 8 × Row → ForEach(allGraphRows)
        │   └── Button[Text(display)]      ← 函数/数字按键
        └── Row (操作按钮)
            ├── Button [绘制]              ← 创建曲线
            └── Button [清空]              ← 清空输入
```

## 10. 与主计算器的协同

主页面顶栏新增两个入口按钮：

```
┌──────────────────────────┐
│              [🖌️] [📚]  │  ← 绘图入口 / 历史入口
│       ...显示区...        │
│       ...按键区...        │
└──────────────────────────┘
```

- 🖌️：`54×54dp` 圆形按钮，点击进入绘图页面，同时重置 `showGraphControls = true`
- 📚：已有历史记录入口

两个入口均为半透明深灰毛玻璃圆形按钮，风格与整体暗黑主题一致。

## 11. 功能对照表（演进总览）

| 能力 | V1 卡片版 | V2 小数点 | V3 暗黑重构 | **V4 绘图版** |
|------|----------|----------|------------|--------------|
| 四则运算 | ✅ | ✅ | ✅ | ✅ |
| 小数点 | ❌ | ✅ | ✅ | ✅ |
| 百分比 | ❌ | ❌ | ✅ | ✅ |
| 历史记录 | ❌ | ❌ | ✅ | ✅ |
| 暗黑主题 | ❌ | ❌ | ✅ | ✅ |
| 毛玻璃按键 | ❌ | ❌ | ✅ | ✅ |
| 幂运算 ^ | ❌ | ❌ | ❌ | **✅** |
| 函数绘图 | ❌ | ❌ | ❌ | **✅** |
| 12 种初等函数 | ❌ | ❌ | ❌ | **✅** |
| 手势平移 | ❌ | ❌ | ❌ | **✅** |
| 手势缩放 | ❌ | ❌ | ❌ | **✅** |
| 多曲线叠加 | ❌ | ❌ | ❌ | **✅** |
| Canvas 渲染 | ❌ | ❌ | ❌ | **✅** |

## 12. 后续优化建议

1. **色轮自动分配**：当前所有曲线均为蓝色 `#2563EB`，可引入预设色板（红/绿/黄/紫/青）自动循环
2. **曲线图例**：在 Canvas 角落绘制色块 + 表达式图例
3. **跟踪坐标**：长按 Canvas 时显示当前触摸点的 (x, y) 值
4. **表达式语法高亮**：输入括号自动配对，函数名以蓝色显示
5. **极坐标模式**：支持 `r = f(θ)` 极坐标绘图
6. **参数方程模式**：支持 `x=f(t), y=g(t)` 参数曲线
7. **曲线持久化**：将 `curves` 保存到 Preferences，卡片重启后恢复已绘制曲线
8. **导出图像**：将 Canvas 内容导出为图片分享
