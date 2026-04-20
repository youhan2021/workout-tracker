---
name: fitness-coach
description: 个人运动追踪教练skill — 低摩擦Telegram输入，训练记录+状态追踪+动态训练建议+日报周报月报。触发：用户发送任何运动/状态/查询相关消息时加载。
---

# fitness-coach

Telegram 低交互场景的个人运动追踪教练。把"像发消息一样记录运动"作为核心目标。

---

## 核心哲学

**3秒记录原则**：用户在任何对话轮次里，发来一条训练或状态消息，agent 应当能在最少回复轮次内完成记录。

---

## 数据文件（统一在 data/ 下）

```
data/
├── plan.json              # 当前训练计划（格式不变）
├── history.json           # 统一历史：训练记录+状态记录+快照引用（升级格式，见下）
├── body_status.json       # 身体基本信息+体重/体脂历史
└── .current_workout.json  # 训练中途进度（临时）
```

### history.json 升级格式

```json
{
  "records": [
    {
      "date": "2026-04-20",
      "type": "workout",
      "workout": {
        "summary": "练了背",
        "exercises": [
          {"name": "坐姿划船", "weight_kg": 25, "sets": [{"reps": 8}, {"reps": 8}, {"reps": 8}]},
          {"name": "高位下拉", "weight_kg": 25, "sets": [{"reps": 8}, {"reps": 8}]}
        ],
        "duration_min": 35,
        "note": "状态不错"
      }
    },
    {
      "date": "2026-04-20",
      "type": "status",
      "status": "累",
      "detail": "昨晚没睡好"
    },
    {
      "date": "2026-04-20",
      "type": "snapshot",
      "snapshot": {
        "week_of": "2026-04-14",
        "workout_count": 4,
        "status_summary": {"累": 2, "正常": 5},
        "plan_version": "v2"
      }
    }
  ]
}
```

---

## 触发判断（每条用户消息都要判断）

当用户发送消息时，按以下顺序判断意图：

### 1. 训练记录类（最高优先级）
关键词/模式：`跑了`、`练了`、`深蹲`、`卧推`、`走了多少步`、`拉伸`、`骑车`、`上半身`、`今天训练了`、`做了.*组`、`Xkg.*X次`

**处理流程：**
1. 提取动作名称、重量、组数、次数、时长
2. 若重量缺失 → 尝试从 plan.json 推断今日计划中的默认重量
3. 若完全无法提取 → 将用户描述作为 `summary` 自由文字保存
4. 调用 `workout.py log` 写入 history.json

**格式灵活接受：**
- `今天跑了40分钟` → `{summary:"跑步40分钟"}`
- `深蹲60kg 5x5，卧推30kg 5x5` → 两个动作分别记录
- `晚上练了腿` → `{summary:"练腿"}`
- `走了8000步` → `{summary:"步行8000步"}`
- `今天拉伸了15分钟` → `{summary:"拉伸15分钟"}`
- `上半身训练` → `{summary:"上半身训练"}`

### 2. 状态记录类
关键词/模式：`累`、`没睡好`、`不想练.*`、`这周只能.*次`、`出差`、`经期`、`恢复`、`不舒服`、`膝盖.*`、`状态`

**处理流程：**
1. 提取状态类型和详情
2. 调用 `workout.py status-log` 写入 history.json
3. 若状态影响训练建议（如"出差"、"经期"、"恢复模式"），触发训练建议调整

**状态类型映射：**
| 用户词 | 存储值 |
|--------|--------|
| 累、很累、疲惫 | `tired` |
| 没睡好、睡眠差 | `poor_sleep` |
| 出差 | `travel` |
| 经期、生理期 | `period` |
| 不想练X、讨厌练X | `avoiding:{部位}` |
| 这周只能练X次 | `frequency_limit:{N}` |
| 恢复、休息一下 | `recovery` |
| 膝盖不舒服、肩膀疼 | `injury:{部位}` |
| 状态不错 | `good` |
| 正常 | `normal` |

### 3. 查询报告类
关键词/模式：`今天练了什么`、`这周练了几次`、`报告`、`周报`、`月报`、`这周为什么执行.*`、`最近计划改了.*`

**处理流程：**
- `今天练了什么` → 查 history.json 中今日 type=workout 记录
- `这周练了几次` → 统计本周 type=workout 数量
- `这周状态` → 查本周 type=status 记录
- `日报` → 调用 `workout.py report today`
- `周报` → 调用 `workout.py report week`
- `月报` → 调用 `workout.py report month`
- `阶段总结` → 调用 `workout.py report summary`

### 4. 计划管理类
关键词/模式：`新计划`、`改训练计划`、`更新计划`、`训练计划.*调整`

**处理流程：**
1. 理解新的训练安排
2. 调用 `workout.py plan set` 保存 plan.json
3. 调用 `workout.py plan version` 追加版本记录到 history

### 5. 身体数据类
关键词/模式：`体重.*kg`、`体脂.*%`、`身高.*cm`

**处理流程：**
调用 `workout.py body update` 或 `workout.py body log`

---

## 训练建议动态调整

### 触发条件
以下状态变化后，agent 应主动或被动更新训练建议：
- 用户说`出差`、`经期`、`恢复模式`、`这周只能练X次` → 降频建议
- 用户说`不想练腿`、`讨厌练背` → 调整部位偏好
- 用户连续3天状态为`tired` → 建议降强度
- 用户连续`poor_sleep` → 建议推迟主训练日

### 调整输出格式
当触发调整时，输出格式：
```
📋 训练建议已调整（v{N}）
原因：{触发原因}
建议：{具体建议}
```

### 方案版本记录
每次调整训练建议，调用 `workout.py plan version <reason>` 记录到 history.json 的 plan_versions 数组。

---

## Quick Actions（适合 Telegram Inline Keyboard ）

提供以下快捷操作选项：

```
[🏋️ 练什么] [📝 记录训练] [📊 查看报告]
[😫 状态记录] [⚖️ 体重记录] [📋 训练计划]
```

---

## 命令一览（供 agent 内部调用脚本）

| 脚本命令 | 功能 |
|----------|------|
| `workout.py log "<date>" '<json>'` | 记录一条训练 |
| `workout.py status-log "<date>" <status> <detail>` | 记录一条状态 |
| `workout.py plan version <reason>` | 记录方案版本变更 |
| `workout.py plan view` | 查看当前计划 |
| `workout.py report today` | 日报 |
| `workout.py report week [date]` | 周报（默认本周） |
| `workout.py report month [date]` | 月报（默认本月） |
| `workout.py report summary` | 阶段总结 |
| `workout.py body update key=value...` | 更新身体数据 |
| `workout.py body log key=value...` | 记录当天体重体脂 |
| `workout.py body view` | 查看身体数据 |
| `workout.py body history [days]` | 体重体脂历史 |
| `workout.py history [days]` | 训练历史（仅训练记录） |
| `workout.py reset-today` | 重置今日 |
| `workout.py reset-date <date>` | 重置指定日 |
| `workout.py start` | 开始交互训练（老功能保留） |

---

## Telegram 低摩擦输入设计原则

1. **不需要确认轮次**：记录完成后直接显示"✅ 已记录：..."
2. **模糊重量可接受**：若用户说"练了腿"无细节，仍可记录为 `{summary:"练腿"}`
3. **同一条消息可混合记录**：如"今天跑了30分钟，状态很累" → 一次性写入训练+状态
4. **查询即时响应**：不要求二次确认，直接查直接回
5. **周报/月报一次性输出**：不分段确认

---

## 报告模板

### 日报
```
📅 2026-04-20 日报
━━━━━━━━━━━━━━━
🏋️ 训练：坐姿划船 25kg×3组 / 高位下拉 25kg×3组
⏱️ 约 35 分钟
😫 状态：累（昨晚没睡好）
📝 备注：—
━━━━━━━━━━━━━━━
```

### 周报
```
📅 周报（4/14 - 4/20）
━━━━━━━━━━━━━━━
🏋️ 训练次数：4次
😫 状态：累×2 / 正常×3 / 出差×1
📉 执行率：4/5 计划日（80%）
⚠️ 问题：周二因出差未训练
📋 方案变更：v2（出差降频）
━━━━━━━━━━━━━━━
```

### 月报
```
📅 月报（4月）
━━━━━━━━━━━━━━━
🏋️ 训练次数：18次
⏱️ 总时长：约 9 小时
😫 状态趋势：上半月累×6 / 下半月正常×8
📉 执行率：18/22（82%）
💪 进步：卧推+5kg，深蹲+10kg
📋 方案变更：3次
━━━━━━━━━━━━━━━
```

---

## 现有数据兼容说明

- `plan.json` 格式**完全不变**，继续使用
- `history.json` **追加兼容**：原有训练记录结构在 `records[].workout` 中，原有格式可直接读取
- `body_status.json` **继续使用**，不变
- `scripts/workout.py` 扩展新命令，不删除原有功能

---

## MVP 第一版实际能力

✅ 以下是本 skill 第一版**实际交付**的能力集：

**输入处理：**
- 自然语言训练记录解析
- 状态记录（7种类型）
- 混合输入（一次消息含训练+状态）

**数据层：**
- history.json 升级格式（含 type=workout/status/snapshot）
- plan.json 继续使用
- body_status.json 继续使用
- 训练方案版本记录

**报告输出：**
- 日报
- 周报
- 月报
- 阶段总结

**训练建议：**
- 基于状态触发动态调整
- 方案版本记录

**Quick Actions：**
- 6个快捷操作选项

❌ 以下为第一版**不做**项（留待后续迭代）：
- 饮食记录
- 热量计算
- 网页端/复杂 UI
- 自动化训练计划生成
