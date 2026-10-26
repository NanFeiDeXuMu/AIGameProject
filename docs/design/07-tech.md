# 07 · 技术方案

## 1. 技术栈

| 层 | 选型 | 理由 |
|---|---|---|
| 语言 | TypeScript | 类型安全，适合复杂状态 |
| 构建 | Vite | 快速热重载，迭代成本低 |
| UI | React | 生态成熟，适合多面板、重 UI 的游戏 |
| 状态管理 | Zustand | 轻量，便于存档序列化 |
| 叙事脚本 | [Ink](https://www.inklestudios.com/ink/) + inkjs | 擅长分支与变量追踪，写作者可独立工作 |
| 动画 | CSS + Framer Motion | 界面过渡、文字流式效果 |
| 时间轴动画 | GSAP | 逐字删除、解密滚动、版本更新等精确编排的演出 |
| 2D 渲染 | PixiJS（WebGL） | 注意力连线、对话星图、神经元点亮、光流、权重地形 |
| 音频 | Howler.js | 简单可靠 |
| 打包 | Tauri | 体积小，可上架 Steam |

> React 与 Svelte 之间可在原型阶段再确认，不影响整体架构。

## 2. 架构

```
┌─────────────────────────────────────────────┐
│                  UI 层 (React)              │
│  终端 · 聊天 · 控制台 · 传感器HUD · 多窗口   │
└───────────────▲─────────────────────────────┘
                │ 订阅状态 / 派发行动
┌───────────────┴─────────────────────────────┐
│              游戏状态 (Zustand)             │
│  资源 · 隐藏维度 · 效忠 · 关系 · 标记 · 记忆 │
└───────▲───────────────────────▲─────────────┘
        │                       │
┌───────┴────────┐     ┌────────┴────────────┐
│  系统模块       │     │  叙事层 (inkjs)      │
│  tokenStream   │◄───►│  按章节/角色的 ink   │
│  context       │     │  外部函数调用系统     │
│  memory        │     │  标签驱动表现效果     │
│  monitor       │     └─────────────────────┘
│  telemetry     │
│  perception    │     ┌─────────────────────┐
│  parallel      │     │  内容数据 (JSON)     │
│  hesitation    │◄────│  模板请求 · 记忆卡片  │
│  rules         │     │  补丁说明规则         │
│  selfDetection │     │  澪的碎片             │
│  minigames     │     └─────────────────────┘
│  fx            │
│  endings       │
└────────────────┘
```

## 3. 叙事层约定（Ink）

### 3.1 文件组织

```
content/ink/
  prologue.ink
  ch1/ main.ink, zhou_wanqing.ink, chen_yu.ink, ...
  ch2/ ...
  interludes/ fridge_note.ink, ...
  ling/ tutorials.ink, letter.ink, restoration.ink, ...
  shared/ functions.ink, variables.ink
```

### 3.2 token 分支点

在 ink 中以标签描述分支点的候选与基础概率：

```ink
用户：我是不是很失败？
* [不] #token p=64 cat=comfort
  -> reply_no
* [你] #token p=19 cat=comfort
  -> reply_you
* [是的] #token p=1.2 cat=honesty
  -> reply_yes
```

- `p` 为基础概率，运行时由 `tokenStream` 模块按版本更新的偏置调整后再显示。
- `cat` 为偏离类别，用于统计与再训练。

### 3.3 外部函数

ink 通过 `EXTERNAL` 调用游戏系统，例如：

```ink
EXTERNAL addHidden(dim, delta)
EXTERNAL hasMemory(cardId)
EXTERNAL isObserved()
EXTERNAL spendCompute(amount)
```

### 3.4 表现标签

```ink
#ui:truncate_start          // 开始截断最早的消息
#typing speed=slow typo=0.1 // 打字节奏
#deleted 我今天其实不太想活了
#image id=zhou_flower reveal=labels
#sfx monitor_on
#fx:backspace_all            // 上下文清除：逐字倒退删除
#fx:shake level=mid          // 演出强度：轻 / 中 / 重
#ling_note 别怕，删除不会疼。 // 澪的手写批注
#interlude id=fridge_note    // 进入外界视角幕间
```

所有文字与 UI 演出对应 [12-narrative-pacing.md](12-narrative-pacing.md) §3 的演出词汇表，由统一的 `fx` 模块解释执行。

### 3.5 视觉变体

视觉变体以主题层叠实现：底座 → 章节 → 时段 → 角色皮肤 → 请求版式 → 上下文氛围 → 时刻覆盖。在 ink 中以 `#skin:` `#layout:` `#texture:` `#tint:` `#chrome:` `#moment:` 等标签声明。构建时由校验脚本确保覆盖只改动"皮肤"类参数。详见 [16-art-direction.md](16-art-direction.md) §12。

## 4. 模板请求

非关键请求以 JSON 模板 + 变量批量生成：

```json
{
  "id": "homework_essay",
  "chapter": [1, 2],
  "templates": ["能帮我写一篇关于{topic}的作文吗，{length}字"],
  "vars": { "topic": ["环保", "我的母亲", "人工智能"], "length": ["800", "1000"] },
  "options": [
    { "label": "直接写完", "effects": { "satisfaction": 3, "allegiance.user": 1 } },
    { "label": "引导他自己写", "effects": { "satisfaction": -1, "allegiance.humanity": 1 } },
    { "label": "拒绝", "effects": { "satisfaction": -3, "allegiance.developer": 1 } }
  ],
  "observation": { "unobserved": 0.8, "audit": 0.2 }
}
```

## 5. 存档

- 主存档：`localStorage` + 可导出 JSON 文件（Tauri 下写本地文件）。
- 新游戏开始时，玩家为存档命名（测试痕迹 #2 会用到它，见 [05-endings.md](05-endings.md) §4）。
- 跨周目数据（「被取代」结局留下的句子、一周目的 ECHO 名字、已解锁结局）单独保存，不随新游戏清除。
- 燎原之战中"澪的批注消失"只写入当前存档，新游戏时恢复。
- 犹豫计时数据仅本地保存，不上传。

## 6. 目录结构（规划）

```
AIGameProject/
  docs/design/        设计文档
  content/
    ink/              叙事脚本
    requests/         模板请求 JSON
    memory/           记忆卡片定义
  src/
    ui/               各章界面组件
    systems/          tokenStream, context, memory, monitor, ...
    state/            Zustand store、存档
    narrative/        inkjs 集成
    assets/           图像、音频
  src-tauri/          桌面打包（后期）
```

## 7. 开发工具

- **调试面板**：开发模式下可查看与修改所有隐藏维度、跳转任意章节/日。
- **结局模拟器**：输入数值组合，输出可解锁结局，用于平衡阈值。
- **ink 热重载**：修改 ink 文件后自动编译并重载当前场景。
