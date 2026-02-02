# 飞书插件多账号支持实现计划

## 概述

本计划描述如何为 `clawdbot-feishu` 插件添加多账号支持，允许用户同时配置多个飞书应用。

参考实现：OpenClaw Discord 插件的多账号模式（`accounts.js`, `config-schema.js`, `provider.js`）。

## 核心设计思路

### 1. 配置合并模式（Merge Mode）

采用 Discord 插件的 **merge 模式**：
- 顶层 `channels.feishu` 作为所有账号的默认配置
- `channels.feishu.accounts` 包含每个账号的特定配置
- 每个账号的最终配置 = 顶层默认 + 账号特定配置（账号特定优先）

```yaml
# 示例配置
channels:
  feishu:
    # 默认配置（所有账号共享）
    domain: feishu
    connectionMode: websocket
    requireMention: true
    historyLimit: 20
    
    # 多账号配置
    accounts:
      work:
        appId: "cli_xxxxx1"
        appSecret: "${FEISHU_WORK_APP_SECRET}"
        domain: feishu
        
      personal:
        appId: "cli_xxxxx2"
        appSecret: "${FEISHU_PERSONAL_APP_SECRET}"
        domain: lark
        dmPolicy: open
        allowFrom: ["*"]
```

### 2. 向后兼容策略

- 如果 `accounts` 字段不存在或为空，使用顶层配置作为 `default` 账号
- 顶层的 `appId`/`appSecret` 视为 `default` 账号的凭据
- 现有配置无需修改即可正常工作

---

## 需要修改的文件列表

### Phase 1: 核心配置和账号解析

| 文件 | 改动说明 | 优先级 |
|------|----------|--------|
| `config-schema.ts` | 添加 `accounts` 字段和账号级 Schema | 🔴 高 |
| `accounts.ts` | 重写账号解析逻辑，支持多账号 merge | 🔴 高 |
| `types.ts` | 添加账号相关类型定义 | 🔴 高 |

### Phase 2: 客户端和连接管理

| 文件 | 改动说明 | 优先级 |
|------|----------|--------|
| `client.ts` | 改成 `Map<accountId, Client>` 多实例缓存 | 🔴 高 |
| `monitor.ts` | 每个账号独立 WebSocket 连接 | 🔴 高 |

### Phase 3: 通道和消息处理

| 文件 | 改动说明 | 优先级 |
|------|----------|--------|
| `channel.ts` | 更新 `config` 适配器支持多账号 | 🟡 中 |
| `bot.ts` | 传递 accountId 到消息处理 | 🟡 中 |
| `send.ts` | 添加 accountId 参数 | 🟡 中 |
| `reply-dispatcher.ts` | 添加 accountId 参数 | 🟡 中 |

### Phase 4: 工具函数

| 文件 | 改动说明 | 优先级 |
|------|----------|--------|
| `docx.ts` | 添加 accountId 参数，获取对应账号的 client | 🟢 低 |
| `wiki.ts` | 添加 accountId 参数 | 🟢 低 |
| `drive.ts` | 添加 accountId 参数 | 🟢 低 |
| `perm.ts` | 添加 accountId 参数 | 🟢 低 |
| `media.ts` | 添加 accountId 参数 | 🟢 低 |
| `directory.ts` | 添加 accountId 参数 | 🟢 低 |
| `probe.ts` | 添加 accountId 参数 | 🟢 低 |
| `onboarding.ts` | 添加 accountId 参数 | 🟢 低 |
| `outbound.ts` | 添加 accountId 参数 | 🟢 低 |

---

## 详细实现步骤

### Phase 1: 核心配置和账号解析

#### 1.1 修改 `config-schema.ts`

**添加账号级 Schema：**

```typescript
// 账号级配置（可覆盖顶层配置）
const FeishuAccountConfigSchema = z
  .object({
    enabled: z.boolean().optional(),
    name: z.string().optional(),  // 账号显示名称
    appId: z.string().optional(),
    appSecret: z.string().optional(),
    encryptKey: z.string().optional(),
    verificationToken: z.string().optional(),
    domain: FeishuDomainSchema.optional(),
    connectionMode: FeishuConnectionModeSchema.optional(),
    webhookPath: z.string().optional(),
    webhookPort: z.number().int().positive().optional(),
    // ... 其他可覆盖的字段
    dmPolicy: DmPolicySchema.optional(),
    allowFrom: z.array(z.union([z.string(), z.number()])).optional(),
    groupPolicy: GroupPolicySchema.optional(),
    groupAllowFrom: z.array(z.union([z.string(), z.number()])).optional(),
    requireMention: z.boolean().optional(),
    groups: z.record(z.string(), FeishuGroupSchema.optional()).optional(),
    historyLimit: z.number().int().min(0).optional(),
    dmHistoryLimit: z.number().int().min(0).optional(),
    renderMode: RenderModeSchema,
    tools: FeishuToolsConfigSchema,
  })
  .strict();

// 更新顶层 Schema，添加 accounts 字段
export const FeishuConfigSchema = z
  .object({
    enabled: z.boolean().optional(),
    // 顶层凭据（向后兼容）
    appId: z.string().optional(),
    appSecret: z.string().optional(),
    encryptKey: z.string().optional(),
    verificationToken: z.string().optional(),
    
    // 多账号配置
    accounts: z.record(z.string(), FeishuAccountConfigSchema.optional()).optional(),
    
    // 默认配置（所有账号共享）
    domain: FeishuDomainSchema.optional().default("feishu"),
    connectionMode: FeishuConnectionModeSchema.optional().default("websocket"),
    // ... 其他字段保持不变
  })
  .strict()
  .superRefine(/* ... */);

export type FeishuAccountConfig = z.infer<typeof FeishuAccountConfigSchema>;
```

#### 1.2 修改 `types.ts`

**添加新类型：**

```typescript
export type FeishuAccountConfig = {
  enabled?: boolean;
  name?: string;
  appId?: string;
  appSecret?: string;
  encryptKey?: string;
  verificationToken?: string;
  domain?: FeishuDomain;
  connectionMode?: FeishuConnectionMode;
  webhookPath?: string;
  webhookPort?: number;
  dmPolicy?: "open" | "pairing" | "allowlist";
  allowFrom?: (string | number)[];
  groupPolicy?: "open" | "allowlist" | "disabled";
  groupAllowFrom?: (string | number)[];
  requireMention?: boolean;
  groups?: Record<string, FeishuGroupConfig | undefined>;
  historyLimit?: number;
  dmHistoryLimit?: number;
  renderMode?: "auto" | "raw" | "card";
  tools?: FeishuToolsConfig;
};

// 更新 ResolvedFeishuAccount
export type ResolvedFeishuAccount = {
  accountId: string;
  enabled: boolean;
  configured: boolean;
  name?: string;
  appId?: string;
  appSecret?: string;
  encryptKey?: string;
  verificationToken?: string;
  domain: FeishuDomain;
  config: FeishuConfig;  // 合并后的完整配置
};
```

#### 1.3 重写 `accounts.ts`

**参考 Discord 的 merge 模式：**

```typescript
import { DEFAULT_ACCOUNT_ID, normalizeAccountId } from "openclaw/plugin-sdk";
import type { ClawdbotConfig } from "openclaw/plugin-sdk";
import type { FeishuConfig, FeishuAccountConfig, ResolvedFeishuAccount, FeishuDomain } from "./types.js";

/**
 * 列出配置中所有的账号 ID
 */
function listConfiguredAccountIds(cfg: ClawdbotConfig): string[] {
  const accounts = (cfg.channels?.feishu as FeishuConfig)?.accounts;
  if (!accounts || typeof accounts !== "object") {
    return [];
  }
  return Object.keys(accounts).filter(Boolean);
}

export function listFeishuAccountIds(cfg: ClawdbotConfig): string[] {
  const ids = listConfiguredAccountIds(cfg);
  if (ids.length === 0) {
    // 向后兼容：无 accounts 配置时返回 default
    return [DEFAULT_ACCOUNT_ID];
  }
  return ids.toSorted((a, b) => a.localeCompare(b));
}

export function resolveDefaultFeishuAccountId(cfg: ClawdbotConfig): string {
  const ids = listFeishuAccountIds(cfg);
  if (ids.includes(DEFAULT_ACCOUNT_ID)) {
    return DEFAULT_ACCOUNT_ID;
  }
  return ids[0] ?? DEFAULT_ACCOUNT_ID;
}

/**
 * 获取指定账号的原始配置
 */
function resolveAccountConfig(
  cfg: ClawdbotConfig,
  accountId: string,
): FeishuAccountConfig | undefined {
  const accounts = (cfg.channels?.feishu as FeishuConfig)?.accounts;
  if (!accounts || typeof accounts !== "object") {
    return undefined;
  }
  return accounts[accountId];
}

/**
 * 合并顶层配置和账号特定配置
 * 账号特定配置优先
 */
function mergeFeishuAccountConfig(
  cfg: ClawdbotConfig,
  accountId: string,
): FeishuConfig {
  const feishuCfg = cfg.channels?.feishu as FeishuConfig | undefined;
  
  // 提取顶层配置（排除 accounts 字段）
  const { accounts: _ignored, ...base } = feishuCfg ?? {};
  
  // 获取账号特定配置
  const account = resolveAccountConfig(cfg, accountId) ?? {};
  
  // 合并：账号配置覆盖顶层配置
  return { ...base, ...account } as FeishuConfig;
}

/**
 * 解析飞书凭据
 */
export function resolveFeishuCredentials(cfg?: FeishuConfig): {
  appId: string;
  appSecret: string;
  encryptKey?: string;
  verificationToken?: string;
  domain: FeishuDomain;
} | null {
  const appId = cfg?.appId?.trim();
  const appSecret = cfg?.appSecret?.trim();
  if (!appId || !appSecret) return null;
  return {
    appId,
    appSecret,
    encryptKey: cfg?.encryptKey?.trim() || undefined,
    verificationToken: cfg?.verificationToken?.trim() || undefined,
    domain: cfg?.domain ?? "feishu",
  };
}

/**
 * 解析完整的账号信息
 */
export function resolveFeishuAccount(params: {
  cfg: ClawdbotConfig;
  accountId?: string | null;
}): ResolvedFeishuAccount {
  const accountId = normalizeAccountId(params.accountId);
  const feishuCfg = params.cfg.channels?.feishu as FeishuConfig | undefined;
  
  // 基础启用状态（顶层）
  const baseEnabled = feishuCfg?.enabled !== false;
  
  // 合并配置
  const merged = mergeFeishuAccountConfig(params.cfg, accountId);
  
  // 账号级启用状态
  const accountEnabled = merged.enabled !== false;
  const enabled = baseEnabled && accountEnabled;
  
  // 解析凭据
  const creds = resolveFeishuCredentials(merged);
  
  return {
    accountId,
    enabled,
    configured: Boolean(creds),
    name: (merged as any).name?.trim() || undefined,
    appId: creds?.appId,
    appSecret: creds?.appSecret,
    encryptKey: creds?.encryptKey,
    verificationToken: creds?.verificationToken,
    domain: creds?.domain ?? "feishu",
    config: merged,
  };
}

/**
 * 列出所有启用且配置完整的账号
 */
export function listEnabledFeishuAccounts(cfg: ClawdbotConfig): ResolvedFeishuAccount[] {
  return listFeishuAccountIds(cfg)
    .map((accountId) => resolveFeishuAccount({ cfg, accountId }))
    .filter((account) => account.enabled && account.configured);
}
```

---

### Phase 2: 客户端和连接管理

#### 2.1 修改 `client.ts`

**改成多实例缓存：**

```typescript
import * as Lark from "@larksuiteoapi/node-sdk";
import type { FeishuDomain, ResolvedFeishuAccount } from "./types.js";

// 从单实例缓存改为 Map 缓存
const clientCache = new Map<string, {
  client: Lark.Client;
  config: { appId: string; appSecret: string; domain: FeishuDomain };
}>();

const wsClientCache = new Map<string, Lark.WSClient>();

function resolveDomain(domain: FeishuDomain) {
  return domain === "lark" ? Lark.Domain.Lark : Lark.Domain.Feishu;
}

/**
 * 创建或获取缓存的飞书客户端
 */
export function createFeishuClient(account: ResolvedFeishuAccount): Lark.Client {
  const { accountId, appId, appSecret, domain } = account;
  
  if (!appId || !appSecret) {
    throw new Error(`Feishu credentials not configured for account "${accountId}"`);
  }

  const cached = clientCache.get(accountId);
  if (
    cached &&
    cached.config.appId === appId &&
    cached.config.appSecret === appSecret &&
    cached.config.domain === domain
  ) {
    return cached.client;
  }

  const client = new Lark.Client({
    appId,
    appSecret,
    appType: Lark.AppType.SelfBuild,
    domain: resolveDomain(domain),
  });

  clientCache.set(accountId, {
    client,
    config: { appId, appSecret, domain },
  });

  return client;
}

/**
 * 创建飞书 WebSocket 客户端（不缓存，每次创建新实例）
 */
export function createFeishuWSClient(account: ResolvedFeishuAccount): Lark.WSClient {
  const { accountId, appId, appSecret, domain } = account;
  
  if (!appId || !appSecret) {
    throw new Error(`Feishu credentials not configured for account "${accountId}"`);
  }

  return new Lark.WSClient({
    appId,
    appSecret,
    domain: resolveDomain(domain),
    loggerLevel: Lark.LoggerLevel.info,
  });
}

/**
 * 创建事件分发器
 */
export function createEventDispatcher(account: ResolvedFeishuAccount): Lark.EventDispatcher {
  return new Lark.EventDispatcher({
    encryptKey: account.encryptKey,
    verificationToken: account.verificationToken,
  });
}

/**
 * 获取指定账号的客户端（如果存在）
 */
export function getFeishuClient(accountId: string): Lark.Client | null {
  return clientCache.get(accountId)?.client ?? null;
}

/**
 * 清除指定账号的客户端缓存
 */
export function clearClientCache(accountId?: string): void {
  if (accountId) {
    clientCache.delete(accountId);
    wsClientCache.delete(accountId);
  } else {
    clientCache.clear();
    wsClientCache.clear();
  }
}
```

#### 2.2 修改 `monitor.ts`

**每个账号独立连接：**

```typescript
import * as Lark from "@larksuiteoapi/node-sdk";
import type { ClawdbotConfig, RuntimeEnv, HistoryEntry } from "openclaw/plugin-sdk";
import { createFeishuWSClient, createEventDispatcher } from "./client.js";
import { resolveFeishuAccount, listEnabledFeishuAccounts } from "./accounts.js";
import { handleFeishuMessage, type FeishuMessageEvent, type FeishuBotAddedEvent } from "./bot.js";
import { probeFeishu } from "./probe.js";

export type MonitorFeishuOpts = {
  config?: ClawdbotConfig;
  runtime?: RuntimeEnv;
  abortSignal?: AbortSignal;
  accountId?: string;
};

// 每个账号独立的 WebSocket 客户端
const wsClients = new Map<string, Lark.WSClient>();
const botOpenIds = new Map<string, string>();

async function fetchBotOpenId(
  account: ResolvedFeishuAccount,
): Promise<string | undefined> {
  try {
    const result = await probeFeishu(account);
    return result.ok ? result.botOpenId : undefined;
  } catch {
    return undefined;
  }
}

/**
 * 启动单个账号的监控
 */
async function monitorSingleAccount(params: {
  cfg: ClawdbotConfig;
  account: ResolvedFeishuAccount;
  runtime?: RuntimeEnv;
  abortSignal?: AbortSignal;
}): Promise<void> {
  const { cfg, account, runtime, abortSignal } = params;
  const { accountId } = account;
  const log = runtime?.log ?? console.log;
  const error = runtime?.error ?? console.error;

  // 获取 bot open_id
  const botOpenId = await fetchBotOpenId(account);
  botOpenIds.set(accountId, botOpenId ?? "");
  log(`feishu[${accountId}]: bot open_id resolved: ${botOpenId ?? "unknown"}`);

  const connectionMode = account.config.connectionMode ?? "websocket";

  if (connectionMode !== "websocket") {
    log(`feishu[${accountId}]: webhook mode not implemented in monitor`);
    return;
  }

  log(`feishu[${accountId}]: starting WebSocket connection...`);

  const wsClient = createFeishuWSClient(account);
  wsClients.set(accountId, wsClient);

  const chatHistories = new Map<string, HistoryEntry[]>();
  const eventDispatcher = createEventDispatcher(account);

  eventDispatcher.register({
    "im.message.receive_v1": async (data) => {
      try {
        const event = data as unknown as FeishuMessageEvent;
        await handleFeishuMessage({
          cfg,
          event,
          botOpenId: botOpenIds.get(accountId),
          runtime,
          chatHistories,
          accountId,  // 传递 accountId
        });
      } catch (err) {
        error(`feishu[${accountId}]: error handling message: ${String(err)}`);
      }
    },
    // ... 其他事件处理保持类似，添加 accountId
  });

  return new Promise((resolve, reject) => {
    const cleanup = () => {
      wsClients.delete(accountId);
      botOpenIds.delete(accountId);
    };

    const handleAbort = () => {
      log(`feishu[${accountId}]: abort signal received, stopping`);
      cleanup();
      resolve();
    };

    if (abortSignal?.aborted) {
      cleanup();
      resolve();
      return;
    }

    abortSignal?.addEventListener("abort", handleAbort, { once: true });

    try {
      wsClient.start({ eventDispatcher });
      log(`feishu[${accountId}]: WebSocket client started`);
    } catch (err) {
      cleanup();
      abortSignal?.removeEventListener("abort", handleAbort);
      reject(err);
    }
  });
}

/**
 * 主入口：启动所有启用账号的监控
 */
export async function monitorFeishuProvider(opts: MonitorFeishuOpts = {}): Promise<void> {
  const cfg = opts.config;
  if (!cfg) {
    throw new Error("Config is required for Feishu monitor");
  }

  const log = opts.runtime?.log ?? console.log;

  // 如果指定了 accountId，只启动该账号
  if (opts.accountId) {
    const account = resolveFeishuAccount({ cfg, accountId: opts.accountId });
    if (!account.enabled || !account.configured) {
      throw new Error(`Feishu account "${opts.accountId}" not configured or disabled`);
    }
    return monitorSingleAccount({
      cfg,
      account,
      runtime: opts.runtime,
      abortSignal: opts.abortSignal,
    });
  }

  // 否则启动所有启用的账号
  const accounts = listEnabledFeishuAccounts(cfg);
  if (accounts.length === 0) {
    throw new Error("No enabled Feishu accounts configured");
  }

  log(`feishu: starting ${accounts.length} account(s): ${accounts.map(a => a.accountId).join(", ")}`);

  // 并行启动所有账号
  await Promise.all(
    accounts.map((account) =>
      monitorSingleAccount({
        cfg,
        account,
        runtime: opts.runtime,
        abortSignal: opts.abortSignal,
      }),
    ),
  );
}

export function stopFeishuMonitor(accountId?: string): void {
  if (accountId) {
    wsClients.delete(accountId);
    botOpenIds.delete(accountId);
  } else {
    wsClients.clear();
    botOpenIds.clear();
  }
}
```

---

### Phase 3: 通道和消息处理

#### 3.1 修改 `channel.ts`

**更新 config 适配器：**

```typescript
// config 部分更新
config: {
  listAccountIds: (cfg) => listFeishuAccountIds(cfg),
  resolveAccount: (cfg, accountId) => resolveFeishuAccount({ cfg, accountId }),
  defaultAccountId: (cfg) => resolveDefaultFeishuAccountId(cfg),
  // ... 其他方法也需要更新以支持 accountId 参数
},

// gateway 部分更新
gateway: {
  startAccount: async (ctx) => {
    const { monitorFeishuProvider } = await import("./monitor.js");
    const account = resolveFeishuAccount({ cfg: ctx.cfg, accountId: ctx.accountId });
    const port = account.config.webhookPort ?? null;
    ctx.setStatus({ accountId: ctx.accountId, port });
    ctx.log?.info(`starting feishu[${ctx.accountId}] (mode: ${account.config.connectionMode ?? "websocket"})`);
    return monitorFeishuProvider({
      config: ctx.cfg,
      runtime: ctx.runtime,
      abortSignal: ctx.abortSignal,
      accountId: ctx.accountId,
    });
  },
},
```

#### 3.2 修改 `bot.ts`

**添加 accountId 参数：**

```typescript
export async function handleFeishuMessage(params: {
  cfg: ClawdbotConfig;
  event: FeishuMessageEvent;
  botOpenId?: string;
  runtime?: RuntimeEnv;
  chatHistories?: Map<string, HistoryEntry[]>;
  accountId?: string;  // 新增
}): Promise<void> {
  const { cfg, event, botOpenId, runtime, chatHistories, accountId } = params;
  
  // 解析账号
  const account = resolveFeishuAccount({ cfg, accountId });
  
  // 使用 account.config 而不是 cfg.channels?.feishu
  const feishuCfg = account.config;
  
  // ... 后续处理中使用 accountId
  
  // 路由时包含 accountId
  const route = core.channel.routing.resolveAgentRoute({
    cfg,
    channel: "feishu",
    accountId,  // 传递 accountId
    peer: { ... },
  });
  
  // ...
}
```

#### 3.3 修改 `send.ts`

**所有发送函数添加 accountId：**

```typescript
export type SendFeishuMessageParams = {
  cfg: ClawdbotConfig;
  to: string;
  text: string;
  replyToMessageId?: string;
  mentions?: MentionTarget[];
  accountId?: string;  // 新增
};

export async function sendMessageFeishu(params: SendFeishuMessageParams): Promise<FeishuSendResult> {
  const { cfg, to, text, replyToMessageId, mentions, accountId } = params;
  
  // 解析账号并获取客户端
  const account = resolveFeishuAccount({ cfg, accountId });
  if (!account.configured) {
    throw new Error(`Feishu account "${account.accountId}" not configured`);
  }
  
  const client = createFeishuClient(account);
  
  // ... 其余逻辑保持不变
}
```

---

### Phase 4: 工具函数

#### 4.1 工具注册时传递 accountId

工具需要知道使用哪个账号。有两种策略：

**策略 A: 工具参数中包含 accountId（推荐）**

```typescript
// doc-schema.ts 更新
export const FeishuDocSchema = Type.Object({
  action: Type.Union([...]),
  doc_token: Type.Optional(Type.String()),
  // ...
  account_id: Type.Optional(Type.String({ description: "Account ID (optional, uses default if not specified)" })),
});

// docx.ts 更新
async execute(_toolCallId, params, context) {
  const p = params as FeishuDocParams;
  const accountId = p.account_id || context?.accountId;
  const account = resolveFeishuAccount({ cfg: api.config!, accountId });
  const client = createFeishuClient(account);
  // ...
}
```

**策略 B: 从执行上下文推断 accountId**

如果 plugin-sdk 支持传递上下文中的 accountId，可以从 context 中获取。

---

## 测试计划

### 单元测试

1. **accounts.ts 测试**
   - `listFeishuAccountIds`: 测试无 accounts、单账号、多账号场景
   - `resolveFeishuAccount`: 测试配置合并逻辑
   - `mergeFeishuAccountConfig`: 测试覆盖优先级
   - `listEnabledFeishuAccounts`: 测试过滤逻辑

2. **client.ts 测试**
   - 多账号客户端缓存独立性
   - 配置变更后缓存更新

### 集成测试

1. **向后兼容测试**
   - 现有无 accounts 配置正常工作
   - 单账号场景等价于 default 账号

2. **多账号测试**
   - 两个账号同时运行
   - 消息正确路由到对应账号
   - 工具使用正确的账号凭据

3. **配置重载测试**
   - 动态添加/删除账号
   - 账号配置变更后重新连接

### 手动测试

1. 配置两个飞书应用（feishu + lark 域）
2. 分别在两个应用的群里 @机器人
3. 验证消息正确处理和回复
4. 验证工具（doc/wiki/drive）使用正确账号

---

## 风险和注意事项

1. **WebSocket 连接数**
   - 每个账号一个 WebSocket 连接
   - 需要监控连接状态和重连逻辑

2. **API 限流**
   - 多账号可能增加 API 调用频率
   - 需要考虑限流策略

3. **凭据安全**
   - 多个 appSecret 的安全存储
   - 推荐使用环境变量：`${FEISHU_XXX_APP_SECRET}`

4. **日志区分**
   - 所有日志需要包含 accountId 前缀
   - 便于调试和监控

---

## 实施时间表

| 阶段 | 预计时间 | 内容 |
|------|----------|------|
| Phase 1 | 0.5 天 | 核心配置和账号解析 |
| Phase 2 | 0.5 天 | 客户端和连接管理 |
| Phase 3 | 0.5 天 | 通道和消息处理 |
| Phase 4 | 0.5 天 | 工具函数更新 |
| 测试 | 0.5 天 | 单元测试 + 集成测试 |

**总计：约 2-2.5 天**

---

## 审批检查项

- [ ] 配置 Schema 设计合理
- [ ] 向后兼容策略无问题
- [ ] 多账号连接管理方案可行
- [ ] 工具的 accountId 传递策略确定
- [ ] 测试计划覆盖充分
