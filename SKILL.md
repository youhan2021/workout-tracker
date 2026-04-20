---
name: workout-tracker
description: 运动规划和记录skill — 导入计划、开始训练、运动历史
---

# workout-tracker

运动规划和记录工具，帮你管理每日训练计划、记录执行情况、查看历史。

## 数据文件

- `data/plan.json` — 运动计划（按周几/序列编排）
- `data/history.json` — 训练历史记录
- `data/body_status.json` — 身体状态（身高/体重/体脂/目标等）
- `data/body_history.json` — 身体状态变化历史

## 命令一览

| 命令 | 说明 |
|------|------|
| `运动计划 <文字描述>` | 文字描述 → LLM转JSON → 自动保存 |
| `运动计划 <json>` | 直接导入JSON格式的运动计划 |
| `运动计划导入 <path>` | 从本地 JSON 文件导入运动计划 |
| `运动计划查看` | 查看当前计划内容 |
| `开始运动` | 根据今天日期判断训练组，开始交互式训练 |
| `跳过此动作` | 当前动作跳过，进入下一个 |
| `运动历史 [天数]` | 输出一张表格显示最近训练记录 |
| `添加记录 <日期> <动作列表>` | 手动添加某日期的训练记录 |
| `重置今日` | 清除今天的训练记录（重新开始） |
| `重置日期 <YYYY-MM-DD>` | 清除指定日期的训练记录 |
| `更新身体数据 <字段>` | 更新身高/体重/体脂/训练目标等基本信息 |
| `记录身体数据 [体重] [体脂]` | 记录今日体重体脂到历史（追踪变化） |
| `查看身体数据` | 查看当前身体状态 |
| `身体数据历史 [天数]` | 查看体重体脂变化历史 |

## 运动计划格式（plan.json）

新版格式为**每组级**（每组独立重量，支持渐进超负荷）：

```json
{
  "name": "我的训练计划",
  "mode": "weekly",
  "schedule": {
    "monday": [
      {
        "name": "坐姿划船",
        "last_set_dropset": false,
        "sets": [
          {"weight_kg": 25, "reps": 8},
          {"weight_kg": 25, "reps": 8},
          {"weight_kg": 25, "reps": 8},
          {"weight_kg": 25, "reps": 8},
          {"weight_kg": 25, "reps": 8, "dropset": true}
        ]
      },
      {
        "name": "高位下拉",
        "sets": [
          {"weight_kg": 25, "reps": 8},
          {"weight_kg": 25, "reps": 8},
          {"weight_kg": 25, "reps": 8}
        ]
      }
    ]
  }
}
```

- `sets` 是数组，每项是一组，可以有不同的 `weight_kg` 和 `reps`
- `last_set_dropset: true` / `dropset: true` 标记该组是 drop set
- `hold_secs` 替代 `reps` 表示等长支撑（如悬挂肩胛）
- 支持旧版双版本格式（向下兼容）

## 训练流程

1. 用户说 `开始运动`
2. 脚本根据日期找到今日训练组
3. 显示训练内容摘要，询问"确认开始？"
4. 确认后，逐一展示动作：
   - 显示：动作名、第几组/共几组
   - 用户回复：做了几个 / `跳过` / `下一组`
   - 记录实际完成数或跳过
5. 所有动作完成后，显示本次训练总结并保存到历史

## 文字描述 → JSON 计划

用户输入 `运动计划 <文字>` 时，agent 会：
1. 理解用户描述的训练内容
2. 生成符合 schema 的 JSON
3. 调用 `运动计划 <json>` 自动保存

示例：
- 用户说：`运动计划 每天早上跑步30分钟，周三做俯卧撑和卷腹各3组`
- agent 生成并保存对应 JSON

## 手动添加历史记录

`添加记录` 命令用于补录漏记的训练：

```
添加记录 2026-04-19 悬挂肩胛下沉 7+3组, 离心引体 2+2组, 坐姿划船 15kg×10×3, 高位下拉 15kg×10×3
```

内部格式（agent 转换后调用脚本）：
```bash
python3 scripts/workout.py add-record "2026-04-19" \
  '[{"name":"悬挂肩胛下沉","weight_kg":null,"sets":[{"reps":7},{"reps":3}]},{"name":"离心引体","weight_kg":null,"sets":[{"reps":2},{"reps":2}]},{"name":"坐姿划船","weight_kg":15,"sets":[{"reps":10},{"reps":10},{"reps":10}]},{"name":"高位下拉","weight_kg":15,"sets":[{"reps":10},{"reps":10},{"reps":10}]}]' \
  '[]'
```

## 脚本使用

```bash
cd ~/.hermes/skills/fitness/workout-tracker

# 导入计划（直接传JSON字符串）
python3 scripts/workout.py plan set '<json字符串>'

# 从文件导入
python3 scripts/workout.py plan import /path/to/plan.json

# 查看当前计划
python3 scripts/workout.py plan view

# 开始训练（交互式）
python3 scripts/workout.py start

# 完成一组（训练中）
python3 scripts/workout.py complete-set 15

# 跳过当前动作
python3 scripts/workout.py skip-action

# 查看历史
python3 scripts/workout.py history
python3 scripts/workout.py history 30

# 重置今日记录
python3 scripts/workout.py reset-today

# 重置指定日期记录
python3 scripts/workout.py reset-date 2026-04-18

# 手动添加历史记录
python3 scripts/workout.py add-record "2026-04-18" \
  '[{"name":"俯卧撑","sets":[15,15,15]}]' '[]'
```

## 内部状态

训练过程中，通过临时文件 `data/.current_workout.json` 跟踪进度。
