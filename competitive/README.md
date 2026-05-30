# 竞品体验采集

> 实际使用竞品时的截图 + 结构化分析。双维度索引：按产品看、按问题看。

---

## 怎么用

### 1. 采集（你做的）

截图丢进 `captures/` 就行，命名建议：`日期-产品-简述.png`

例：
- `2026-05-31-keling-prompt-guide.png`
- `2026-05-31-jimeng-style-picker.png`
- `2026-06-01-midjourney-v6-variation.png`

命名不严格，能认出来就行。

### 2. 标注（告诉 Claude）

丢完截图后告诉我：
- 这是哪个产品的
- 你觉得它在解决什么问题
- （可选）你觉得好在哪 / 不好在哪

我来写结构化记录，更新索引。

### 3. 查看

- **想看某个产品的所有设计** → `by-product/` 下找对应产品
- **想看某个问题各家怎么解** → `by-problem/` 下找对应 topic
- **文档站上也能看** → 导航里"竞品观察"

---

## 目录结构

```
competitive/
├── README.md              ← 你在看的这个
├── captures/              ← 截图原文件（直接丢这里）
├── by-product/            ← 按产品汇总
│   ├── keling.md          ← 可灵的所有观察
│   ├── jimeng.md          ← 即梦
│   ├── midjourney.md      ← Midjourney
│   ├── meitu.md           ← 美图
│   ├── runway.md          ← Runway
│   └── ...
└── by-problem/            ← 按问题域汇总（对应 topic）
    ├── intent-understanding.md
    ├── iteration-precision.md
    ├── style-consistency.md
    ├── speed-and-waiting.md
    └── ...
```

---

## 每条记录的格式

```markdown
## [产品名] - [功能/设计点简述]

![截图描述](../captures/文件名.png)

- **产品**：xxx
- **关联 topic**：意图理解 / 迭代精准性 / ...
- **解决什么问题**：用户遇到了什么困难
- **怎么解的**：产品用了什么设计/策略
- **好在哪**：为什么这个解法有效
- **不好在哪**：（如果有）局限性或 tradeoff
- **对抖音的启发**：能借鉴什么、不能照搬什么
```
