# Cycle 1+2 Demo —— 可执行实现 Plan
## 《乱码与卡牌 / Static & Signal》极简 Vertical Slice

> 本文档是 GDD v0.1 的下一层。GDD 描述"是什么"，本文档描述"怎么做、做多大、用什么数"。
> 目标：基于本文档可以直接产出一个**单文件 HTML demo**，玩家能在 10–12 分钟内走完两个 cycle 的完整 game loop，**亲身感受到核心机制 + 验证 loop 是否成立**。
> 版本：v0.2 · 日期：2026-04-25
>
> v0.1 → v0.2 主要改动：范围从单 cycle 扩到 2 cycle；Cycle 2 加入 Discard/Keep/Fuse 决策与合卡机制——因为没有第二轮就看不到"deck evolution"，而它是这个游戏整个机制论点的核心。

---

## 0. 范围与显式决策

**核心原则：极限压缩玩法，但保证完整 game loop 闭环。**

| 项 | 决策 |
|---|---|
| 包含的流程 | Tutorial（精简）→ Cycle 1（听课→作业→反馈）→ **Deck Decision**（挤卡三选一）→ Cycle 2（听课→作业→反馈）→ "End of demo" |
| **不含** | 第 3 cycle 及以后、生活片段插画、跨多次的 deck 演化（只演化 1 次）、多结局演出、存档、音效、配乐 |
| 技术栈 | 单文件 HTML + 原生 JS + CSS。无构建、无依赖、无框架 |
| 美术 | 占位级别：CSS 形状 + Unicode 符号。所有"插画位"用色块占位 |
| 文案语言 | 导师/反馈用英文，UI 控件用英文，记忆文本中文 |
| 时长目标 | 单次完整游玩 10–12 分钟 |
| 不做的视觉 | 拼贴美术、复杂动画、音效（留 hook，不实做） |

**显式不做、留给完整 vertical slice 的：**
- Cycle 3 + 期末作业 + 三结局演出
- Cycle 间的"生活片段"插画与母语卡激活提示
- 注意力池跨多 cycle 的连续扩容曲线
- 多周目数据对比（Ending C vs Ending A 的累积成本提示）

**为什么必须包含 Cycle 2**：deck evolution（GDD §4.4）是整个游戏机制论点的核心——"成长伴随失去"、"双文化认同需要主动付出"。这件事在单 cycle 里**不可能**发生，只能描述。Cycle 2 是它最便宜的体现场所。

---

## 1. 符号集 / Symbol Vocabulary

固定 **6 个符号**（GDD 范围 6–8，取下限以减少手牌组合复杂度）：

| ID | Glyph | Native 风格 | Translated 风格 |
|---|---|---|---|
| `circle`   | ◯ | 手绘圆，外圈带毛边/小圆点装饰 | 几何精确圆，单线 |
| `square`   | □ | 略歪的方框，四角有小弧 | 等边正方，直角硬朗 |
| `triangle` | △ | 等腰三角，底边波浪 | 等边三角，刻度感 |
| `diamond`  | ◇ | 菱形+内部斜线纹理 | 菱形+内部网格 |
| `cross`    | ✚ | 加粗十字，端点圆头 | 细十字，端点尖角 |
| `star`     | ☆ | 五角星，描边手抖 | 五角星，等分对称 |

**混合卡 / Fusion** 视觉：母语外框 + 翻译内核（如 ◯ 包 □），后续 SVG 处理。Demo 阶段用双色边框区分。

> Demo 阶段：用 Unicode 字形 + CSS `font-family` 区分（母语用 serif/手写体，翻译用 sans-mono，融合用双重边框）。

---

## 2. 卡牌设计

### 2.1 母语卡（Native Cards · 玩家初始牌库 8 张，全部固定）

每张卡：`{ id, name, left, right, origin:'native', memory }`

| # | name | left | right | memory（鼠标悬停显示） |
|---|---|---|---|---|
| 1 | 外婆的厨房     | circle   | diamond  | 外婆把刚剥好的橘子塞进我手里，说"凉，你戴帽子" |
| 2 | 雨夜回家       | triangle | circle   | 巷子湿，路灯黄，钥匙转了三下才进对孔 |
| 3 | 旧课本边角     | square   | triangle | 高一数学第 43 页画了一只翅膀坏掉的鸟 |
| 4 | 春节的红包     | cross    | diamond  | 压岁钱被妈妈收走，半夜又偷偷塞回我枕头底下 |
| 5 | 楼下小卖部     | circle   | star     | 一块钱三颗的水果糖，糖纸我留了一抽屉 |
| 6 | 方言口头禅     | triangle | cross    | 外婆说"莫法"——这个词在普通话里没有完全对应的翻译 |
| 7 | 高考前夜的台灯 | square   | diamond  | 凌晨两点台灯下，最后一道压轴题画了三遍辅助线 |
| 8 | 第一次坐火车   | star     | square   | 窗外的田后退得比我想象的快 |

**符号分布检查**：6 个符号每个至少出现 2 次，不出现"某个符号在母语卡里完全没有"的死局。

### 2.2 翻译卡（Translated Cards · 程序生成）

听课阶段每捕获 1 个符号，事后生成 1 张翻译卡。

生成规则：
- 卡的 `left = capturedSymbol`
- 卡的 `right = ` 从 6 个符号中**带权随机**（已捕获过的符号权重 ×2，避免出现"两端都是没听过的符号"的不真实感）
- `name`：从下面的池子里抽（与符号无关，只为了"学术外语感"）：
  - "Compositional Logic" / "Negative Space" / "Material Tension" / "Frame and Field"
  - "Iteration" / "Critical Distance" / "Picture Plane" / "Surface Reading"
- `memory: null`（翻译卡没有记忆文本，悬停显示卡名 + "Translated · meaning unclear"）

> 翻译卡的 right 端是"随机"的，让玩家感觉"这是我捕获的，但带出来一些我也不完全控制的东西"——对应外语习得的副作用。

### 2.3 混合卡（Fusion Cards · Cycle 2 决策可生成）

通过 Fuse 操作生成（详见 §5.6）。

- `{ id, name, left, right, origin:'fusion', sourceMem }`
- `name`：母语卡名 + " / " + 翻译卡名（例："雨夜回家 / Negative Space"）
- 视觉：双重边框（外母语+内翻译）
- **匹配特殊性**：链条匹配时仍按 left/right 符号匹配（不变），但出牌评分时**额外触发** Legibility 与 Voice 的双倍计入（详见 §6.1）

---

## 3. 听课阶段 / Listening Phase

### 3.1 时间与流速

| Cycle | token 总数 | 流速 | 注意力池 | brief 关键位数 |
|---|---|---|---|---|
| 1 | 40 | 800 ms / token | **4** | 5 |
| 2 | 50 | 800 ms / token | **5**（+1） | **6**（+1） |

> 设计意图：Cycle 2 注意力 +1（"在适应"），但关键位 +1（"复杂度涨得更快"）——保持 GDD §4.1 "成长伴随失去"的形状。两个 cycle 都数学上**注定漏掉至少 1 个关键位**。

### 3.2 注意力池规则

- 每次点击 token 消耗 1 点
- 池子见底时，token 仍可悬停查看，但不可捕获，光标变 `not-allowed`
- 已滚过的 token 在屏幕保留 ~3s 后淡出，淡出后无法再点击

### 3.3 导师独白脚本

#### Cycle 1 脚本（40 token，5 关键位）

```
pos  symbol     isKey
 1   null       -      "Today,"
 2   circle     -
 3   null       -
 4   square     YES    ★ Brief 关键位 #1
 5   triangle   -
 6   null       -
 7   diamond    -
 8   null       -
 9   circle     YES    ★ Brief 关键位 #2
10   null       -
11   cross      -
12   square     -
13   null       -
14   star       YES    ★ Brief 关键位 #3
15   null       -
16   triangle   -
17   diamond    -
18   null       -
19   null       -
20   square     -
21   cross      YES    ★ Brief 关键位 #4
22   null       -
23   circle     -
24   triangle   -
25   null       -
26   diamond    YES    ★ Brief 关键位 #5
27–40  …杂项填充，无 key
```

#### Cycle 2 脚本（50 token，6 关键位）

关键位密度略升，开头节奏更快（导师"今天讲的更深"）：

```
pos  symbol     isKey
 1   triangle   YES    ★ #1   ←开场就 key（玩家容易没准备好）
 2   null       -
 3   square     -
 4   diamond    YES    ★ #2
 5   null       -
 6   cross      -
 7   circle     YES    ★ #3
 8   null       -
 9   square     -
10   star       -
11   null       -
12   triangle   -
13   cross      YES    ★ #4
14   null       -
15   diamond    -
16   circle     -
17   null       -
18   star       YES    ★ #5
19   square     -
20   null       -
21   triangle   -
22   diamond    YES    ★ #6
23–50  …杂项填充
```

**关键位放在 1, 4, 7, 13, 18, 22**——前段密集，后段稀疏，模拟"刚开始懵了一下，后面慢慢追上"的体感。

### 3.4 捕获后的处理

被捕获的 token：
1. 立即从乱码变成对应符号（◯ □ △ 等）
2. 飞入屏幕右侧 "Captured" 侧栏，按时间顺序排列
3. 同步进入 `STATE.capturedPositions` 与 `STATE.capturedSymbols`
4. 听课结束后，每个被捕获符号 → 生成 1 张翻译卡

### 3.5 听课结束

token 都滚完后，自动转 Brief Reveal 页面。**不允许跳过**。

---

## 4. Brief 生成

听课结束后：

1. 找出所有 `isKey: true` 的 position 的 symbol → `keyBriefSymbols`（Cycle 1 长度 5，Cycle 2 长度 6）
2. 对每个 brief 位检查：玩家是否点击捕获过那个 position？
   - 是 → Brief 该位显示明确符号
   - 否 → Brief 该位显示 `?`
3. 同时秘密保存 `briefSecretAnswers`：所有位置（包括 ?）背后的真实 symbol，用于猜测判定

> 严守：判定标准是"是否捕获了那个 position"，不是"是否捕获了那个 symbol"。否则机制变成"集邮符号"。

---

## 5. 作业阶段 / Assignment Phase

### 5.1 链条规模

- Cycle 1 链长 = **5 张**
- Cycle 2 链长 = **6 张**（同步 Brief 长度）

### 5.2 手牌

- 母语卡（剩余的）+ 当前所有翻译卡 + 混合卡（如有）
- 全部正面朝上展开在底部，可横向滚动

### 5.3 出牌交互

1. Brief 常驻屏幕顶部
2. 链条 slot 横向排列，初始全空
3. 玩家点击手牌 → 放置到下一个空 slot
4. 出牌时检查：当前卡的 `left` 是否等于上一张卡的 `right`？
   - 是 → 安静连接
   - 否 → 弹出二次确认："Force this connection? It won't fit cleanly. (Coherence -25)"
     - 玩家可选 "Force" 或 "Cancel"
     - Force → 视觉显示一道裂纹/红线，`mismatches += 1`
5. 已出的牌可撤回最后一张（按 Z 或点 "Undo"），不可中间修改
6. 卡牌**不可翻转**（demo 简化，左永远朝左，右永远朝右）

### 5.4 提交

全部出完后，"Submit" 按钮亮起。点击后：
- 短转场（1.5 s）：手牌淡出，链条变成抽象色块拼贴
- 进入反馈页

### 5.5 已出过的卡的命运

- **不消耗**：出牌只是"在这次作业里使用"，作业结束后卡回到牌库
- 这是 Cycle 1 → Cycle 2 牌库连续性的基础

### 5.6 Deck Decision Screen（Cycle 1 反馈后触发）

**触发条件**：Cycle 1 反馈结束 → 进入 Cycle 2 听课**之前**插入此屏幕。

**逻辑前提**：
- 牌库上限 = **10 张**
- Cycle 1 后，牌库 = 8 母语 + N 翻译卡（N = Cycle 1 捕获数，0–4）
- 如果 8 + N ≤ 10 → 系统**强制**额外发 (10 + 1 - 8 - N) = (3 - N) 张随机翻译卡，让玩家进入"超额"状态
- 即：**进入 Decision 时牌库始终 = 11 张**（超出上限 1 张），必须丢/合 1 张

> 这个 hack 是为了让 demo **稳定触发**决策——否则太依赖玩家 Cycle 1 听得多不多。文案上呈现为："More words have come in. Your deck can hold 10."

#### Decision UI 与三选项

屏幕中央显示一句话："Your deck is over capacity. Choose what stays."
下方平铺所有 11 张卡（可点选）。底部三按钮：

##### A. Discard
- 选中 1 张卡 → 点 Discard → 该卡永久移除
- 母语卡被丢弃时，触发**记忆闪回**：屏幕短暂变暗（1 s），中央显示该卡的 `memory` 文本，淡出
- 翻译卡被丢弃时，无演出（设计：母语卡的失去才有重量）
- 完成后进入 Cycle 2

##### B. Keep
- 选中 1 张**翻译卡** → 点 Keep → 该翻译卡被丢弃，没有演出
- 文案："Refuse this addition. Your words stay yours, for now."
- 仅翻译卡可被 Keep（即"拒绝接受这张外语表达"）
- **代价**：Cycle 2 注意力池 −1（从 5 降到 4）。在 UI 上明示这条
- 完成后进入 Cycle 2

##### C. Fuse
- 选中 2 张卡（必须 1 张母语 + 1 张翻译）→ 点 Fuse
- **条件**：两张卡必须**至少共享 1 个符号**（不论左右端），否则按钮 disabled，提示 "These don't share enough to fuse."
- 生成 1 张混合卡：
  - left = 母语卡未共享的那个端的符号
  - right = 翻译卡未共享的那个端的符号
  - 例：母语 [circle, diamond] + 翻译 [circle, star] → 共享 circle → 混合卡 [diamond, star]
  - 如果两张卡共享两个符号（罕见）→ 混合卡 [母语 left, 翻译 right]
- **代价**：当前 Cycle 1 反馈展示的 Coherence 在 demo 数据中 −10（仅 internal，玩家无感）；UI 上明示："Fusing takes more from you than you think."
- 完成后进入 Cycle 2

#### 设计意图守护

- **Discard 最快**：1 次点击 + 1 次 Discard 完成。代价是失去（演出体现）
- **Keep 第二快**：1 次点击 + 1 次 Keep。代价是 Cycle 2 听课更难
- **Fuse 最累**：2 次点击 + 检查共享条件 + 找配对。代价是回报最大但需要主动找
- 三个选项**等量呈现**：按钮同样大小、同样颜色、无任何"推荐"hint
- 不出现"are you sure?" 二次确认——决策一旦做出立即生效，避免玩家"试错"心态

---

## 6. 评分系统

**计算分数，但不向玩家显示数字**——只用文字反馈传达。

### 6.1 三个轴的公式（0–100）

**Legibility**
```
明确格命中：每命中 1 个 brief 明确符号 +20
"?"格猜测：
  - 玩家放在该 slot 的卡若任一端命中 briefSecretAnswers[i] → +15
  - 任一端都不命中 → −10
翻译卡占比：链中翻译卡数 × 5
混合卡奖励：链中每张混合卡 +10（双倍计入）
合计裁剪到 [0, 100]
```

**Voice**
```
基础：母语卡数 × 12
首位置加分：slot 1 是母语卡 +15
末位置加分：最后 slot 是母语卡 +15
"心头位"加分：链条中位（slot 3 或 4）是母语卡 +10
混合卡奖励：链中每张混合卡 +15（双倍计入）
合计裁剪到 [0, 100]
```

**Coherence**
```
100 - mismatches × 25
最低 0
```

> 混合卡的双倍计入是 GDD §4.4 中"评分中同时加 Legibility 和 Voice"的直接落地。

### 6.2 反馈文本生成

把每个轴的分数映射到三档（高 ≥70 / 中 40–69 / 低 <40），导师叫 **Dr. Hollow**。

| 轴 | 高 | 中 | 低 |
|---|---|---|---|
| Legibility | "I can see what you're trying to say. The argument lands." | "Some parts come through. Others… I'm filling in for you." | "I'm honestly not sure I follow this." |
| Voice | "There's something distinctly yours in this. Keep that." | "I catch glimpses of a perspective. Bring it forward more." | "It feels assembled rather than expressed. Where are you in this?" |
| Coherence | "It holds together as a piece." | "The transitions are uneven, but the whole stands." | "The pieces don't quite fit. Reconsider how they connect." |

**Cycle 2 反馈额外有一段"对比句"**（如果 demo 允许）：
- 如果 Cycle 2 比 Cycle 1 同轴升档：`"Compared to last time — [轴名] is sharper now."`
- 如果同档：`""`（不说）
- 如果降档：`"Compared to last time — [轴名] feels less here."`

> 严守：不显示分数、不给星级、不出现 "Good job!"、Cycle 2 反馈结束**不暗示哪个结局更好**。

---

## 7. UI / Screen Flow

```
[Title] → [Tutorial ×3] → [Lecture #1] → [Brief Reveal #1] → [Assignment #1] → [Submit] → [Feedback #1]
                                                                                                ↓
                                                                                      [Deck Decision]
                                                                                                ↓
[End of demo · Restart]  ←  [Feedback #2]  ←  [Submit]  ←  [Assignment #2]  ←  [Brief Reveal #2]  ←  [Lecture #2]
```

### Screen 设计

- **Title**: 居中标题 "Static & Signal" + 副标题 "a vertical slice · 2 cycles" + "Begin"
- **Tutorial**: 3 张静态卡片，左右翻页，可 Skip
- **Lecture**: 顶部导师插画占位 + 中部滚动文本流 + 底部 Attention Pool 指示 + 右下 Captured 侧栏
- **Brief Reveal**: "Today's brief, as you understood it:" + 5/6 个大格（明确 or `?`）+ "Begin assignment"
- **Assignment**: Brief 顶部常驻 + 链条 slot + 手牌底部 + Undo / Submit
- **Submit 转场**: 1.5 s，链条变色块拼贴
- **Feedback**: 三段反馈文本，纵向排列，无分数、无图标
- **Deck Decision**（Cycle 1 反馈后）: 11 张卡平铺 + 三按钮 [Discard] [Keep] [Fuse] + 状态提示
- **End of demo**: 简短文字 + Restart 按钮

---

## 8. 数据结构（建议的 JS shape）

```js
const SYMBOLS = ['circle','square','triangle','diamond','cross','star'];

const NATIVE_DECK = [/* 8 张，对应 §2.1 */];

const LECTURE_SCRIPT = {
  cycle1: [/* 40 项 */],
  cycle2: [/* 50 项 */],
};

const STATE = {
  screen: 'title',                       // title|tutorial|lecture|brief|assign|submit|feedback|decision|end
  currentCycle: 1,                       // 1 | 2

  // 听课阶段
  attentionPool: 4,
  capturedPositions: new Set(),
  capturedSymbols: [],

  // 牌库（持久跨 cycle）
  deck: [...NATIVE_DECK],                // 当前完整牌库（演化中）
  translatedThisCycle: [],               // 本 cycle 新生成的翻译卡（决策前用）

  // 作业阶段
  brief: [],                             // 长度 5/6，元素是 symbolId 或 null
  briefSecretAnswers: [],                // 长度 5/6，"?" 位置背后的真实符号
  hand: [],
  chain: [],
  mismatches: 0,

  // 评分
  scores: { legibility:0, voice:0, coherence:0 },
  feedback: { l:'', v:'', c:'' },
  prevScores: null,                      // 用于 Cycle 2 对比句

  // Decision 阶段
  decisionTaken: null,                   // 'discard' | 'keep' | 'fuse'
};
```

---

## 9. 实现里程碑

按"先跑通整个 loop、再美化"。每个 milestone 结束有一个可玩状态。

| M | 范围 | 估时 | 可玩状态 |
|---|---|---|---|
| M1 | 全部 screen 静态切换骨架，按钮跳转 | 0.5d | 走完 UI 流程，无逻辑 |
| M2 | 听课流：token 滚动、点击捕获、Pool 消耗（仅 Cycle 1） | 1d | Cycle 1 听课完整 |
| M3 | Brief 生成 + 作业出牌 + 匹配检查 + 强制连接（Cycle 1） | 1d | Cycle 1 全流程闭环（含反馈） |
| M4 | Deck Decision 屏：11 卡平铺、三按钮逻辑、Discard/Keep/Fuse 各自处理 | 1d | 决策能跑通，状态正确传递 |
| M5 | Cycle 2 听课 + 作业 + 反馈（复用 Cycle 1 代码 + 参数化） | 0.5d | Demo 端到端打通 |
| M6 | 评分对比句、记忆闪回演出、CSS 收尾 | 0.5d | 适合发 2–3 人试玩 |
| M7 | 数值微调（流速、池容量、阈值），文字打磨 | 0.5d | 准备好提交作品集/给评估者看 |

**总：5 个工作日**（不含美术资源替换）

> v0.1 是 3.5d；v0.2 多出来的 1.5d 主要在 M4（Deck Decision 是 demo 中最贵的单屏）和 Cycle 2 端到端调通。

---

## 10. Vertical Slice 验证标准

GDD §9 的 5 项，本 demo 目标 **稳定触发 4 项**（v0.1 只能触发 3 项）：

- [x] **至少一次"该死，没听到那个词"** —— Cycle 1 流速 + 关键位分布达成
- [x] **至少一次"该花注意力还是省着"的犹豫** —— Pool 容量 4 < 关键位 5
- [x] **第一次作业后理解了某个之前模糊的符号** —— Cycle 1 Brief 反馈了 ?, 玩家在 Cycle 2 听课时会回想"那个符号原来是 X"
- [x] **面对 Discard/Keep/Fuse 决策时停下来想了几秒** —— Deck Decision 屏专门为此设计，三选项等量呈现 + 母语卡丢弃有演出
- [⚠️] 反馈出现时主动想要再读一遍出牌选择 —— 反馈不显示分数 → 玩家会回看链。但 demo 没有"撤回反馈"按钮，只能在心里看，需要 playtest 观察

如果 5 项中能稳定触发 ≥4 项，机制原型就成立了，可以扩到 Cycle 3 + 多结局。

**额外的"loop 流畅度"自检**：
- [ ] 玩家在 Cycle 1 反馈页能预期"接下来还有事要做"，不觉得游戏已经结束
- [ ] Decision 后进入 Cycle 2，玩家**记得**自己刚做了什么决定（如果 Decision 与 Cycle 2 之间没有过渡，体验会断）
- [ ] Cycle 2 反馈页让玩家产生"如果我刚才那张卡选别的会怎样"的反思

---

## 11. 数值调试清单（playtest 时优先调这些）

按影响力排序：

1. **token 流速**（800 ms / token）→ 太快变成"无法理解"，太慢丢失"赶不上"。每次只调 ±100 ms
2. **Cycle 1 → Cycle 2 注意力扩容幅度**（+1）→ 关系到玩家是否感觉到"在适应"
3. **关键位增长速度**（5 → 6）→ 关系到"复杂度涨得更快"是否成立。如果玩家 Cycle 2 抓得反而更全，就要 +2
4. **牌库上限**（10）→ 决定决策强度。改 12 的话玩家可能不需要做艰难选择
5. **Fuse 共享符号要求**（≥1 共享）→ 太松所有人都能合，太紧没人合。可备选改成"共享符号必须出现在两张卡的相同端"
6. **Keep 的代价**（Cycle 2 Pool −1）→ 太轻 Keep 变最优，太重玩家不敢用
7. **mismatch 扣分**（25）→ 1 次扣到中档，2 次扣到低档。可能偏重，备选 15
8. **猜对/猜错的分差**（+15 / −10）→ 太大变赌徒游戏，太小没意思

**不要先调的**：母语卡内容、反馈文本——叙事层，机制没稳定前不动。

---

## 12. 文件结构建议

```
/SOTS游戏/
  cycle12_demo.html          ← 单文件入口
  /assets/                    ← 后续放 SVG 符号、插画
    placeholder/             ← 当前用色块占位
```

单文件结构：
- `<style>`：所有 CSS（按 screen 分块，约 250 行）
- `<body>`：每个 screen 用 `<section data-screen-active>` 控制显隐
- `<script>`：
  - 顶部常量：`SYMBOLS`、`NATIVE_DECK`、`LECTURE_SCRIPT.cycle1/cycle2`、文本反馈表
  - `STATE` 对象
  - 每个 screen 的 `render()` + 事件 handler
  - `startLecture(cycleNum)` / `submitChain()` / `computeScores()` / `triggerDecision()` / `applyFusion(a,b)` 等

**避免**：引入 React / Vue / Svelte。原生 JS 足够，且更便于改数值时不重构。

---

## 13. 接下来一步

收到本文档反馈后：
1. 你挑战/调整本文档的决策（特别是 §0 范围、§3.3 脚本、§5.6 决策代价、§6 公式）
2. 定稿后我直接产出 `cycle12_demo.html`，按 §9 的 milestone 顺序写
3. **建议**：M3（Cycle 1 闭环）和 M5（Demo 端到端打通）各暂停一次，让你试玩 → 调数值 → 再继续。这两个 checkpoint 比所有 playtest 设计都重要
