# KAIROS-05：最佳实践

> KAIROS 系统的使用建议和常见模式。

## 任务设计原则

### 1. 幂等性

确保任务可以安全地重复执行：

```typescript
// ❌ 不好：重复执行会产生重复数据
const task1 = {
  command: 'echo "data" >> results.txt',
}

// ✅ 好：使用追加模式或检查存在性
const task2 = {
  command: 'grep -q "data" results.txt || echo "data" >> results.txt',
}
```

### 2. 超时设置

始终为任务设置合理的超时：

```typescript
const task = {
  command: 'npm install',
  options: {
    timeout: 5 * 60 * 1000,  // 5 分钟
  },
}
```

### 3. 错误处理

使用 `set -e` 确保错误被捕获：

```typescript
const task = {
  command: 'set -e; npm run build; npm run test',
}
```

## 常见模式

### 1. 定期健康检查

```typescript
const healthCheck = {
  expression: '*/5 * * * *',  // 每 5 分钟
  command: 'curl -f http://localhost:3000/health || alert-admin',
  name: 'Health Check',
}
```

### 2. 数据库备份

```typescript
const backup = {
  expression: '0 2 * * *',  // 每天凌晨 2 点
  command: 'pg_dump $DATABASE_URL | gzip > backup-$(date +%Y%m%d).sql.gz',
  name: 'Database Backup',
}
```

### 3. 日志轮转

```typescript
const logRotate = {
  expression: '0 0 * * 0',  // 每周日凌晨
  command: 'find /var/log -name "*.log" -mtime +30 -delete',
  name: 'Log Rotation',
}
```

### 4. 依赖更新

```typescript
const updateDeps = {
  expression: '0 9 * * 1',  // 每周一上午 9 点
  command: 'npm update && npm audit fix',
  name: 'Update Dependencies',
}
```

## 性能优化

### 1. 批量操作

```typescript
// ❌ 不好：多个独立任务
const tasks = [
  { expression: '0 * * * *', command: 'process-a.sh' },
  { expression: '0 * * * *', command: 'process-b.sh' },
  { expression: '0 * * * *', command: 'process-c.sh' },
]

// ✅ 好：合并为一个任务
const task = {
  expression: '0 * * * *',
  command: 'process-a.sh && process-b.sh && process-c.sh',
}
```

### 2. 并行执行

```typescript
// 使用 & 并行执行独立任务
const task = {
  expression: '0 * * * *',
  command: 'task1.sh & task2.sh & task3.sh & wait',
}
```

### 3. 资源限制

```typescript
const task = {
  command: 'nice -n 10 ionice -c2 -n7 heavy-task.sh',
}
```

## 监控和日志

### 1. 结构化日志

```typescript
const task = {
  command: `
    echo "[$(date -Iseconds)] Starting task"
    if run-task; then
      echo "[$(date -Iseconds)] Task completed successfully"
    else
      echo "[$(date -Iseconds)] Task failed with exit code $?"
    fi
  `,
}
```

### 2. 通知集成

```typescript
const task = {
  command: `
    if ! backup-db; then
      curl -X POST $SLACK_WEBHOOK -d '{"text":"Backup failed!"}'
      exit 1
    fi
  `,
}
```

### 3. 执行记录

```typescript
const task = {
  command: `
    {
      echo "Start: $(date)"
      my-command
      echo "End: $(date)"
    } > /var/log/task-$(date +%Y%m%d-%H%M%S).log 2>&1
  `,
}
```

## 故障排查

### 1. 调试 Cron 表达式

```bash
# 使用 crontab.guru 验证
# https://crontab.guru/#0_0_*_*_0

# 本地测试下次执行时间
node -e "
const cron = require('cron');
const job = new cron.CronJob('0 0 * * 0', () => {});
console.log('Next execution:', job.nextDate().toString());
"
```

### 2. 环境变量

```typescript
// Cron 任务有独立的环境
const task = {
  command: `
    export PATH="/usr/local/bin:$PATH"
    export NODE_ENV="production"
    my-command
  `,
}
```

### 3. 权限问题

```typescript
// 确保有执行权限
const task = {
  command: `
    if [ ! -x ./script.sh ]; then
      chmod +x ./script.sh
    fi
    ./script.sh
  `,
}
```

## 安全建议

1. **最小权限原则**：使用专用账户运行任务
2. **敏感数据**：使用环境变量或密钥管理服务
3. **审计日志**：记录所有任务执行
4. **网络限制**：限制远程触发器的来源 IP

## 本章小结

本系列完整介绍了 KAIROS 系统：
1. 系统介绍
2. 定时任务
3. Cron 任务
4. 远程触发器
5. 最佳实践

KAIROS 是 Claude Code 的重要扩展，为 AI 工具提供了时间感知能力。
