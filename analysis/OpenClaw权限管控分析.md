# OpenClaw 权限管控分析报告

## 概述

本文档分析 OpenClaw 的权限管控机制，特别是网关重启、配置修改等敏感操作的权限控制方案。

## 一、当前架构分析

### 1.1 认证机制

根据源码分析，OpenClaw 当前的认证机制包括：

1. **速率限制** (`src/gateway/auth-rate-limit.ts`)
   - 滑动窗口速率限制器
   - 按 {scope, clientIp} 跟踪失败尝试
   - 默认：10次尝试/60秒，锁定5分钟
   - 回环地址 (127.0.0.1 / ::1) 默认豁免

2. **认证范围**
   - `AUTH_RATE_LIMIT_SCOPE_DEFAULT` - 默认
   - `AUTH_RATE_LIMIT_SCOPE_SHARED_SECRET` - 共享密钥
   - `AUTH_RATE_LIMIT_SCOPE_DEVICE_TOKEN` - 设备令牌
   - `AUTH_RATE_LIMIT_SCOPE_HOOK_AUTH` - Webhook 认证

### 1.2 权限管控现状

**当前问题：**
1. 缺乏细粒度的权限控制
2. 网关重启、配置修改等敏感操作没有权限限制
3. 所有通过认证的用户都可以执行所有操作
4. 没有基于角色的访问控制 (RBAC)

## 二、权限管控改造方案

### 2.1 整体架构设计

```
┌─────────────────────────────────────────────────────────┐
│                     请求入口                              │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│              认证层 (Authentication)                      │
│  - Token验证                                            │
│  - 速率限制                                              │
│  - IP白名单                                              │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│              授权层 (Authorization)                       │
│  - 角色识别 (Role Identification)                        │
│  - 权限检查 (Permission Check)                           │
│  - 策略评估 (Policy Evaluation)                          │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│              操作层 (Action Execution)                    │
│  - 网关控制 (start/stop/restart)                        │
│  - 配置管理 (read/write)                                │
│  - 账户管理 (add/remove)                                │
│  - 插件管理 (install/uninstall)                          │
└─────────────────────────────────────────────────────────┘
```

### 2.2 角色定义 (RBAC)

| 角色名称 | 权限范围 | 描述 |
|---------|---------|------|
| **admin** | 全部权限 | 系统管理员，可执行所有操作 |
| **operator** | 网关操作 | 可重启网关、查看状态 |
| **configurator** | 配置管理 | 可修改配置文件 |
| **user** | 基本使用 | 仅可使用基本功能，无法修改系统 |
| **viewer** | 只读 | 仅可查看状态和配置 |

### 2.3 权限矩阵

| 操作 | admin | operator | configurator | user | viewer |
|-----|-------|----------|--------------|------|--------|
| 网关启动 | ✅ | ✅ | ❌ | ❌ | ❌ |
| 网关停止 | ✅ | ✅ | ❌ | ❌ | ❌ |
| 网关重启 | ✅ | ✅ | ❌ | ❌ | ❌ |
| 查看网关状态 | ✅ | ✅ | ✅ | ✅ | ✅ |
| 读取配置 | ✅ | ✅ | ✅ | ✅ | ✅ |
| 修改配置 | ✅ | ❌ | ✅ | ❌ | ❌ |
| 管理账户 | ✅ | ❌ | ❌ | ❌ | ❌ |
| 管理插件 | ✅ | ❌ | ❌ | ❌ | ❌ |
| 查看日志 | ✅ | ✅ | ✅ | ❌ | ❌ |

### 2.4 配置文件设计

**配置文件位置：** `~/.openclaw/permissions.json`

```json
{
  "version": "1.0",
  "roles": {
    "admin": {
      "permissions": ["*"]
    },
    "operator": {
      "permissions": [
        "gateway.start",
        "gateway.stop",
        "gateway.restart",
        "gateway.status",
        "config.read",
        "logs.read"
      ]
    },
    "configurator": {
      "permissions": [
        "gateway.status",
        "config.read",
        "config.write",
        "logs.read"
      ]
    },
    "user": {
      "permissions": [
        "gateway.status",
        "config.read"
      ]
    },
    "viewer": {
      "permissions": [
        "gateway.status",
        "config.read"
      ]
    }
  },
  "users": {
    "ou_1d08222c409d14daf6b446df529cfe7a": {
      "role": "admin",
      "created_at": "2026-03-31T00:00:00Z"
    }
  },
  "policies": {
    "require_confirmation_for": [
      "gateway.stop",
      "gateway.restart",
      "config.write"
    ],
    "session_timeout_minutes": 30,
    "ip_allowlist": []
  }
}
```

## 三、实施建议

### 3.1 第一阶段：基础权限框架

1. 创建权限配置文件结构
2. 实现角色和权限定义
3. 添加用户-角色映射
4. 实现基础权限检查中间件

### 3.2 第二阶段：敏感操作控制

1. 网关重启、停止操作加权限检查
2. 配置修改操作加权限检查
3. 添加操作确认机制
4. 实现操作审计日志

### 3.3 第三阶段：高级功能

1. 支持自定义角色
2. 支持细粒度权限控制
3. 添加临时权限提升
4. 实现权限继承

## 四、关键代码修改点

### 4.1 权限检查中间件

```typescript
// src/gateway/permission-middleware.ts
interface PermissionCheckOptions {
  requiredPermission: string;
  requireConfirmation?: boolean;
}

function checkPermission(
  userId: string,
  options: PermissionCheckOptions
): PermissionCheckResult {
  // 1. 获取用户角色
  // 2. 检查角色权限
  // 3. 返回检查结果
}
```

### 4.2 网关命令权限控制

```typescript
// src/daemon/commands/gateway.ts
async function restartGateway(userId: string) {
  const check = checkPermission(userId, {
    requiredPermission: "gateway.restart",
    requireConfirmation: true
  });
  
  if (!check.allowed) {
    throw new Error("Permission denied");
  }
  
  // 执行重启
}
```

## 五、总结

OpenClaw 当前缺乏细粒度的权限管控机制，建议通过 RBAC 模型来实现权限控制，分三个阶段逐步实施，确保系统安全性的同时保持良好的用户体验。
