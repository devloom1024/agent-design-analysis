# acpx — 工具调用与权限

## 架构特点

使用 **TTY 交互模式**——默认在终端中直接询问用户，也支持预定义策略。

## 权限策略默认值

```typescript
type PermissionMode = 'approve-all' | 'approve-reads' | 'deny-all';
// 默认: 未指定时走交互模式

type NonInteractivePermissionPolicy = 'deny' | 'fail';
// 默认: 'deny'
```

## resolvePermissionRequest() 完整决策树

```
1. options 为空 → cancelled()
2. mode === 'approve-all':
   → 选 allow_once/allow_always → 直接返回
3. mode === 'deny-all':
   → 选 reject_once/reject_always 或 cancelled()
4. 自动推断 toolKind (inferToolKind):
   - read/search → auto approved (approve-reads 逻辑)
   - edit/delete/move/execute/fetch/think/other → 需交互
5. 非 TTY 环境 (!canPromptForPermission):
   - nonInteractivePolicy === 'fail' → PermissionPromptUnavailableError
   - nonInteractivePolicy === 'deny' → 选 reject 或 cancelled
6. TTY 交互 → promptForToolPermission() → y/N 询问
```

## inferToolKind() 推断规则

```typescript
title.includes('read') || 'cat'    → 'read'
title.includes('search')/find/grep → 'search'
title.includes('write')/edit/patch → 'edit'
title.includes('delete')/remove    → 'delete'
title.includes('move')/rename      → 'move'
title.includes('run')/execute/bash → 'execute'
title.includes('fetch')/http/url   → 'fetch'
title.includes('think')            → 'think'
其他                               → 'other'
```

## client.ts 中的 handlePermissionRequest

```typescript
private async handlePermissionRequest(params: RequestPermissionRequest):
  1. session 正在取消 → cancelled
  2. resolvePermissionRequest(params, permissionMode, nonInteractivePermissions)
  3. classifyPermissionDecision → 记录 PermissionStats
  4. 返回 RequestPermissionResponse
```

## classifyPermissionDecision()

```typescript
selectedOption.kind === 'allow_once' || 'allow_always' → 'approved'
selectedOption.kind === 'reject_once' || 'reject_always' → 'denied'
未选择 → 'cancelled'
```

## ACP 协议权限选项

```typescript
interface PermissionOption {
  optionId: string;
  name: string;
  kind: 'allow_once' | 'allow_always' | 'reject_once' | 'reject_always';
}
```

## PermissionStats 统计

```typescript
type PermissionStats = {
  requested: number;
  approved: number;
  denied: number;
  cancelled: number;
};
```

## 权限相关退出码

```typescript
EXIT_CODES.PERMISSION_DENIED = 5
```

## 配置文件中的权限设置

```json
{
  "defaultPermissions": "approve-reads",
  "nonInteractivePermissions": "deny"
}
```
