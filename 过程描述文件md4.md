# 过程描述文件 4：代码质量重构与进制换算功能

## 1. 版本概述

本次迭代聚焦两个方向：(1) 全量级的数据结构现代化——将核心引擎中的 `Record` 字典替换为 ES6 `Map`，消除索引签名访问的潜在风险；(2) 新增**进制换算**模块——支持二进制/八进制/十进制/十六进制之间的双向实时转换，使应用功能矩阵扩展至计算机科学基础领域。

## 2. 核心引擎重构：Record → Map

### 2.1 变更动机

原代码使用 `Record<string, number>` / `Record<string, Function>` 作为运算符优先级和处理器的存储结构：

```typescript
// 旧版（索引访问，无类型安全）
const OPERATORLEVELS: Record<string, number> = { '+': 0, '-': 0, '*': 1, '/': 1, '^': 2 };
const result = OPERATORLEVELS[someString];  // 可能为 undefined
```

`Record` 的索引签名 (`[key: string]`) 对任意字符串返回 `number | undefined`，编译器无法在编译期校验键名的合法性。运行时若传入未定义的运算符，静默返回 `undefined` 进而导致后续解析错误，排查困难。

### 2.2 Map 化改造

```typescript
// 新版（Map 显式存取，类型安全）
const operatorLevels = new Map<string, number>([
  ['+', 0], ['-', 0], ['*', 1], ['/', 1], ['^', 2]
]);

function isPrioritized(first: string, second: string): boolean {
  const firstLevel = operatorLevels.get(first) ?? 0;
  const secondLevel = operatorLevels.get(second) ?? 0;
  // ...
}
```

| 对比维度 | Record | Map |
|----------|--------|-----|
| 键类型安全 | 任意 string 可索引 | `.get(key)` 显式查询 |
| 缺失处理 | 隐式 `undefined` | `??` 默认值显式处理 |
| 迭代顺序 | 插入顺序不保证 | 严格插入顺序 |
| 内存效率 | 原型链开销 | 纯哈希结构 |
| 键存在检测 | `key in record` | `map.has(key)` 语义清晰 |

### 2.3 受影响的全局常量

| 旧名称 | 新名称 | 类型 |
|--------|--------|------|
| `OPERATORLEVELS` | `operatorLevels` | `Map<string, number>` |
| `OPERATORHANDLERS` | `operatorHandlers` | `Map<string, CalcHandler>` |
| `CONSTANTS` | `constantsMap` | `Map<string, number>` |

命名风格同步从全大写常量转为 camelCase，符合 ArkTS/TypeScript 现代编码规范。

### 2.4 运算符处理器安全调用

```typescript
// 旧版（直接索引，无防护）
const result = OPERATORHANDLERS[element](secondStackElement, firstStackElement);

// 新版（Map.get + null guard）
const handler = operatorHandlers.get(token);
if (handler) {
  const res = handler(a, b);
  stack.push(res.length > 15 ? Number.parseFloat(res).toExponential() : res);
} else {
  stack.push('0');  // 降级处理
}
```

`handler` 可能为 `undefined` 的情况现在被显式 `if` 分支处理，杜绝了 `undefined is not a function` 运行时崩溃。

### 2.5 方法签名规范化

所有方法添加了显式返回类型注解：

```typescript
drawGraph(): void { ... }
zoomGraph(factor: number): void { ... }
appendGraphChar(ch: string): void { ... }
onInputValue = (value: string): void => { ... }
```

ArkTS 编译器对 `@State` 组件的方法有严格的返回值推断要求，显式注解避免了隐式 `any` 推导警告。

## 3. 新功能：进制换算（Base Conversion）

### 3.1 功能入口

主页面顶栏新增 🔄 按钮（54×54dp 圆形，半透明毛玻璃），点击进入进制换算页面：

```
主页面 → [🔄] → 进制换算页面
              → [🖌️] → 函数绘图页面
              → [📚] → 历史记录页面
```

应用现已覆盖 4 个功能页面：计算器 / 历史记录 / 函数绘图 / 进制换算。

### 3.2 页面 UI 布局

```
┌──────────────────────────────┐
│  [←]   进制换算               │
├──────────────────────────────┤
│  ┌────────────────────────┐  │
│  │ BIN  10101011          │  │  ← 输入面板 1 (橘色值，可点击)
│  └────────────────────────┘  │
│  ┌────────────────────────┐  │
│  │ DEC  171               │  │  ← 输入面板 2 (橘色值，可点击)
│  └────────────────────────┘  │
│                              │
│  [7] [8] [9] [AC]           │  ← 数字键盘（红底 AC）
│  [4] [5] [6] [⌫]            │
│  [1] [2] [3] [F]            │  ← 含十六进制字母键
│ [00] [0] [.] [E]            │
│  [A] [B] [C] [D]            │  ← 全 HEX 字母行
└──────────────────────────────┘
```

### 3.3 交互模型

**双面板设计**：上下两个面板分别代表一种进制。点击任一面板将其设为「当前编辑面板」（`activeInput`），键盘输入影响该面板的值，另一个面板实时显示转换结果。

```
用户点击上面板 (activeInput = 1)
  → 在上面板输入 "FF"（十六进制）
  → 下面板实时显示 "255"（十进制）

用户点击下面板 (activeInput = 2)
  → 在下面板输入 "100"（十进制）
  → 上面板实时显示 "1100100"（二进制）
```

**进制切换**：点击面板弹出半屏选择器（Action Sheet 风格），列出 BIN/OCT/DEC/HEX 四选项，当前项以橙色 ✓ 标记。

```
┌──────────────────────┐
│   取消   选择进制      │
├──────────────────────┤
│   二进制 BIN         │
│ ───────────────────  │
│   八进制 OCT         │
│ ───────────────────  │
│   十进制 DEC    ✓    │  ← 当前选中
│ ───────────────────  │
│   十六进制 HEX       │
└──────────────────────┘
```

### 3.4 转换引擎

#### 3.4.1 任意进制 → 十进制 `parseToDecimal()`

```
输入: "1A.8" (base=16)
  ├─ 整数部分:  1×16¹ + 10×16⁰ = 26
  ├─ 小数部分:  8×16⁻¹ = 0.5
  └─ 结果: 26.5
```

算法支持**小数部分解析**，通过负指数累加实现任意进制小数的十进制转换。支持**负号前缀**检测。

#### 3.4.2 十进制 → 任意进制 `decimalToBase()`

```
输入: 26.5 (toBase=16, precision=8)
  ├─ 整数部分:  26 → 反复除16取余 → "1A"
  ├─ 小数部分:  0.5 → 反复乘16取整 → "8"
  └─ 结果: "1A.8"
```

整数部分使用短除法（除基取余，逆序输出），小数部分使用倍增法（乘基取整，顺序输出），精度上限 8 位小数防无限循环。

#### 3.4.3 字符合法性校验 `isValidCharForBase()`

```typescript
function isValidCharForBase(ch: string, base: number): boolean {
  const idx = HEX_CHARS.indexOf(ch.toUpperCase());  // 在 "0123456789ABCDEF" 中查找
  if (idx === -1) return false;                     // 非十六进制字符
  if (idx >= base) return false;                    // 超出当前进制范围
  if (base === 10 && ch === '.') return true;       // 十进制允许小数点
  if (ch === '.' && base !== 10) return false;      // 非十进制不允许小数点
  return true;
}
```

| 进制 | 允许字符 | 示例 |
|------|----------|------|
| BIN (2) | 0, 1 | `1011` |
| OCT (8) | 0-7 | `173` |
| DEC (10) | 0-9, . | `42.5` |
| HEX (16) | 0-9, A-F | `FF00A3` |

只有十进制允许小数点，其他进制输入小数点时静默忽略，避免非法字符进入表达式。

#### 3.4.4 实时转换 `convertBase()`

每次按键后自动触发转换流水线：

```
sourceValue (当前编辑面板的字符串)
    │
    ▼ parseToDecimal(value, sourceBase)
decimal (中间十进制数值)
    │
    ▼ decimalToBase(decimal, targetBase)
targetStr (目标面板的字符串)
```

双向均可作为输入源——用户可在任一进制面板输入，另一面板自动同步。

### 3.5 进制页面键盘

```
Row 1: 7    8    9    AC       ← AC 使用红色半透明底 (rgba(253,59,48,0.8))
Row 2: 4    5    6    ⌫
Row 3: 1    2    3    F        ← F 键（十六进制）
Row 4: 00   0    .    E        ← E 键 + 小数点
Row 5: A    B    C    D        ← 全十六进制字母键
```

键盘通过 `@Builder Btn()` 方法统一构建，`AC` 键以红色半透明底（`rgba(253, 59, 48, 0.8)`）作为破坏性操作的视觉警示。

`00` 键在进制换算场景中仅追加字符 `"00"`（不做计算器场景下的智能纠错），保持输入值的原始性。

### 3.6 进制切换时的值过滤

切换进制时，当前输入值可能包含新进制不合法的字符，通过 `filterValue()` 过滤：

```
BIN "1102" 切换为 DEC → 过滤 '2' → "110"
HEX "FF" 切换为 BIN → 'F' 超出范围 → ""
```

### 3.7 新增状态字段

```typescript
@State showBaseConversion: boolean = false;  // 进制页面可见性
@State input1Base: number = 2;               // 面板 1 进制（默认 BIN）
@State input1Value: string = '';             // 面板 1 数值
@State input2Base: number = 10;              // 面板 2 进制（默认 DEC）
@State input2Value: string = '';             // 面板 2 数值
@State activeInput: number = 1;             // 当前编辑面板 (1 或 2)
@State showBasePicker: boolean = false;      // 进制选择器可见性
@State pickerTarget: number = 1;            // 选择器目标面板 (1 或 2)
```

## 4. 导航体系升级

主页面顶栏从 2 按钮扩展为 3 按钮：

```
[🔄] [🖌️] [📚]
 进制   绘图   历史
```

| 按钮 | 图标 | 目标页面 | 额外操作 |
|------|------|----------|----------|
| 🔄 | Unicode 循环箭头 | `showBaseConversion` | `resetBaseInputs()` 重置为 BIN↔DEC |
| 🖌️ | 画笔 | `showGraph` | `showGraphControls = true` 展开输入面板 |
| 📚 | 书籍 | `showHistory` | — |

## 5. 构建器（@Builder）方法

本次引入两个 ArkTS `@Builder` 装饰器方法，实现 UI 模板复用：

| Builder | 用途 | 复用场景 |
|---------|------|----------|
| `Btn(char, type)` | 进制键盘按键 | 5 行键盘 × 4 键 = 20 次复用 |
| `BaseOption(base, label)` | 进制选择器选项 | BIN/OCT/DEC/HEX 4 次复用 |

`@Builder` 是 ArkTS 的轻量级模板语法，允许在组件内定义可复用的 UI 片段，避免重复编写样式代码，同时享受声明式 UI 的响应式更新。

## 6. 组件层级树（新增部分）

```
showBaseConversion = true
└── Column (全屏, #000000)
    ├── Row (顶栏)
    │   ├── Button [←]
    │   └── Text '进制换算'
    ├── Column (面板 1)
    │   └── Column (可点击)
    │       ├── Text (进制名, 16fp, 白)
    │       └── Text (数值, 32fp, #FF9500)
    ├── Column (面板 2)
    │   └── Column (可点击)
    │       ├── Text (进制名)
    │       └── Text (数值)
    └── Column (键盘区, layoutWeight:1)
        ├── Row [7, 8, 9, AC]       ← Btn()
        ├── Row [4, 5, 6, DEL]      ← Btn()
        ├── Row [1, 2, 3, F]        ← Btn()
        ├── Row [00, 0, ., E]       ← Btn()
        └── Row [A, B, C, D]        ← Btn()

showBasePicker = true
└── Column (遮罩)
    └── Column (选择器, #1C1C1E)
        ├── Row (标题栏)
        │   ├── Button [取消]
        │   └── Text '选择进制'
        ├── BaseOption(2, '二进制 BIN')     ← @Builder
        ├── Divider
        ├── BaseOption(8, '八进制 OCT')
        ├── Divider
        ├── BaseOption(10, '十进制 DEC')
        ├── Divider
        └── BaseOption(16, '十六进制 HEX')
```

## 7. 功能演进总览

| 能力 | V1 | V2 | V3 | V4 | **V5** |
|------|----|----|----|----|--------|
| 四则运算 | ✅ | ✅ | ✅ | ✅ | ✅ |
| 小数点 | ❌ | ✅ | ✅ | ✅ | ✅ |
| 百分比 / 00 | ❌ | ❌ | ✅ | ✅ | ✅ |
| 历史记录 | ❌ | ❌ | ✅ | ✅ | ✅ |
| 暗黑主题 + 毛玻璃 | ❌ | ❌ | ✅ | ✅ | ✅ |
| 函数绘图 (12种) | ❌ | ❌ | ❌ | ✅ | ✅ |
| Canvas 手势交互 | ❌ | ❌ | ❌ | ✅ | ✅ |
| 幂运算 ^ | ❌ | ❌ | ❌ | ✅ | ✅ |
| **进制换算 (BIN/OCT/DEC/HEX)** | ❌ | ❌ | ❌ | ❌ | **✅** |
| **Record → Map 重构** | ❌ | ❌ | ❌ | ❌ | **✅** |
| **@Builder 模板复用** | ❌ | ❌ | ❌ | ❌ | **✅** |

## 8. 后续优化建议

1. **进制扩展**：支持 3/5/6/7/9 等非标准进制，将 `baseNames` Map 改为动态生成
2. **位运算模式**：增加 AND/OR/XOR/NOT/SHIFT 位运算面板
3. **IP 地址换算**：点分十进制 ↔ 十六进制 ↔ 二进制 IP 格式转换
4. **颜色码换算**：Hex `#FF9500` ↔ RGB `255,149,0` ↔ HSL 颜色空间互转
5. **复制功能**：长按数值面板复制转换结果到剪贴板
6. **键盘音效**：为十六进制字母键（A-F）添加区别于数字键的音效
