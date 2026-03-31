# KAIROS-03：Cron 任务实现

> 深入分析 KAIROS 的 Cron 任务支持。

## Cron 表达式支持

KAIROS 使用标准 Cron 表达式语法：

```
* * * * * *
│ │ │ │ │ │
│ │ │ │ │ └─ 星期几 (0-6, 0=周日)
│ │ │ │ └─── 月份 (1-12)
│ │ │ └───── 日期 (1-31)
│ │ └─────── 小时 (0-23)
│ └───────── 分钟 (0-59)
└─────────── 总是匹配 (*)
```

### 常用示例

| 表达式 | 说明 |
|--------|------|
| `0 * * * *` | 每小时整点 |
| `0 0 * * *` | 每天午夜 |
| `0 0 * * 0` | 每周日凌晨 |
| `0 0 1 * *` | 每月 1 号午夜 |
| `*/5 * * * *` | 每 5 分钟 |
| `0 9-17 * * 1-5` | 工作日 9-17 点每小时 |

## Cron 工具实现

```typescript
interface CronInput {
  expression: string
  command: string
  name?: string
  timezone?: string
}

async function createCronJob(input: CronInput): Promise<string> {
  // 1. 验证 Cron 表达式
  const parsed = parseCronExpression(input.expression)

  // 2. 计算下次执行时间
  const nextExecution = getNextExecution(parsed, input.timezone)

  // 3. 创建任务
  const task: ScheduledTask = {
    id: generateId(),
    name: input.name || `Cron: ${input.expression}`,
    command: input.command,
    trigger: {
      type: 'once',
      timestamp: nextExecution,
    },
    cronExpression: input.expression,  // 保存用于重新调度
  }

  // 4. 注册任务
  const taskId = await scheduleTask(task)

  return taskId
}
```

## Cron 解析器

```typescript
interface CronExpression {
  minute: CronField
  hour: CronField
  day: CronField
  month: CronField
  weekday: CronField
}

interface CronField {
  type: 'wildcard' | 'exact' | 'range' | 'list' | 'step'
  values?: number[]
  min?: number
  max?: number
  step?: number
}

function parseCronExpression(expr: string): CronExpression {
  const parts = expr.trim().split(/\s+/)

  if (parts.length !== 5) {
    throw new Error('Invalid Cron expression')
  }

  return {
    minute: parseField(parts[0], 0, 59),
    hour: parseField(parts[1], 0, 23),
    day: parseField(parts[2], 1, 31),
    month: parseField(parts[3], 1, 12),
    weekday: parseField(parts[4], 0, 6),
  }
}

function parseField(field: string, min: number, max: number): CronField {
  // * - 通配符
  if (field === '*') {
    return { type: 'wildcard' }
  }

  // */n - 步长
  if (field.startsWith('*/')) {
    const step = parseInt(field.slice(2))
    return { type: 'step', step, min, max }
  }

  // n-m - 范围
  if (field.includes('-')) {
    const [start, end] = field.split('-').map(Number)
    return { type: 'range', values: range(start, end) }
  }

  // n,m,o - 列表
  if (field.includes(',')) {
    const values = field.split(',').map(Number)
    return { type: 'list', values }
  }

  // n - 精确值
  const value = parseInt(field)
  return { type: 'exact', values: [value] }
}
```

## 下次执行时间计算

```typescript
function getNextExecution(cron: CronExpression, timezone = 'UTC'): number {
  const now = new Date()
  const baseTime = timezone === 'UTC' ? now : convertToTimezone(now, timezone)

  // 从下一分钟开始检查
  const candidate = new Date(baseTime)
  candidate.setMinutes(candidate.getMinutes() + 1, 0, 0)

  // 最多向前查找 4 年（闰年边界）
  for (let i = 0; i < 4 * 366 * 24 * 60; i++) {
    if (matchesCron(cron, candidate)) {
      return candidate.getTime()
    }
    candidate.setMinutes(candidate.getMinutes() + 1)
  }

  throw new Error('No matching time found in 4 years')
}

function matchesCron(cron: CronExpression, date: Date): boolean {
  return (
    matchesField(cron.minute, date.getMinutes()) &&
    matchesField(cron.hour, date.getHours()) &&
    matchesField(cron.day, date.getDate()) &&
    matchesField(cron.month, date.getMonth() + 1) &&
    matchesField(cron.weekday, date.getDay())
  )
}
```

## Cron 任务重新调度

```typescript
// 执行完成后重新调度
async function rescheduleCronTask(taskId: string, task: ScheduledTask) {
  if (!task.cronExpression) {
    return  // 非 Cron 任务
  }

  // 1. 解析 Cron 表达式
  const cron = parseCronExpression(task.cronExpression)

  // 2. 计算下次执行时间
  const nextExecution = getNextExecution(cron)

  // 3. 创建新任务
  const newTask: ScheduledTask = {
    ...task,
    id: generateId(),
    trigger: {
      type: 'once',
      timestamp: nextExecution,
    },
  }

  await scheduleTask(newTask)
}
```

## 时区支持

```typescript
function convertToTimezone(date: Date, timezone: string): Date {
  // 使用 Intl API 转换时区
  const formatter = new Intl.DateTimeFormat('en-US', {
    timeZone: timezone,
    year: 'numeric',
    month: 'numeric',
    day: 'numeric',
    hour: 'numeric',
    minute: 'numeric',
    second: 'numeric',
  })

  const parts = formatter.formatToParts(date)

  return new Date(
    parts.find(p => p.type === 'year')?.value,
    parts.find(p => p.type === 'month')?.value - 1,
    parts.find(p => p.type === 'day')?.value,
    parts.find(p => p.type === 'hour')?.value,
    parts.find(p => p.type === 'minute')?.value,
    parts.find(p => p.type === 'second')?.value,
  )
}
```

## 本章小结

本章介绍了 KAIROS 的 Cron 任务实现：
- Cron 表达式语法
- Cron 解析器
- 下次执行时间计算
- 自动重新调度
- 时区支持

## 下一章

第 4 章将介绍远程触发器。
