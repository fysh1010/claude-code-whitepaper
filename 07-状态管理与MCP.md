# 第七章：状态管理与 MCP

## 7.1 状态管理概述

Claude Code 采用自定义的 Zustand 风格状态管理系统，结合 React Context 实现跨组件状态共享。这种设计既保证了状态更新的高性能，又提供了清晰的 API 接口。状态管理是整个应用的数据中心，所有的 UI 组件都依赖于状态管理来获取和更新数据。

状态管理的核心设计原则包括：细粒度订阅，组件只订阅需要的状态片段，避免不必要的重渲染；不可变更新，使用 Object.is 比较状态变化，确保更新效率；类型安全，完整的 TypeScript 类型定义，包含深度不可变类型；以及模块化设计，状态按照功能领域划分到不同的子状态中。

MCP（Model Context Protocol）是 Claude Code 与外部工具和服务通信的协议。MCP 系统允许 Claude Code 连接外部服务器，扩展其能力范围。通过 MCP，Claude Code 可以访问文件系统、数据库、API 等外部资源，而不需要将这些功能直接内置到代码中。

## 7.2 Store 实现

### 7.2.1 Store 类型定义

**文件**: `src/state/store.ts` (第 4-8 行)

Store 是状态管理的核心接口，提供了状态读取、写入和订阅的基本功能。

```typescript
export type Store<T> = {
  getState: () => T;           // 获取当前状态
  setState: (updater: (prev: T) => T) => void;  // 更新状态
  subscribe: (listener: Listener) => () => void;  // 订阅状态变化
};

// Listener 类型
type Listener = () => void;
```

这个简单的接口实现了 Zustand 的核心功能，足以满足应用的需求。

### 7.2.2 创建 Store

```typescript
// 创建 Store 的基本模式
function createStore<T>(initialState: T): Store<T> {
  let state = initialState;
  const listeners = new Set<Listener>();

  return {
    getState: () => state,

    setState: (updater) => {
      const nextState = updater(state);
      if (nextState !== state) {
        state = nextState;
        listeners.forEach(listener => listener());
      }
    },

    subscribe: (listener) => {
      listeners.add(listener);
      return () => listeners.delete(listener);
    },
  };
}
```

这种实现使用了 Set 来存储监听器，确保每个监听器只会被添加一次，并且在组件卸载时能够正确移除。

### 7.2.3 React 集成

**文件**: `src/state/AppState.tsx` (第 142-179 行)

为了在 React 组件中使用 Store，需要使用 useSyncExternalStore Hook。

```typescript
export function useAppState<R>(selector: (state: AppState) => R): R {
  const store = useAppStore();
  const get = useCallback(() => selector(store.getState()), [selector, store]);

  // useSyncExternalStore 确保 SSR 安全和一致的状态
  return useSyncExternalStore(
    store.subscribe,
    get,
    get
  );
}
```

useSyncExternalStore 是 React 18 引入的 Hook，专门用于订阅外部数据源。它确保了在服务器端渲染和客户端hydration时的状态一致性。

## 7.3 应用状态结构

### 7.3.1 AppState 类型

**文件**: `src/state/AppStateStore.ts` (第 89-452 行)

AppState 是整个应用的核心状态类型，包含了所有需要在组件间共享的数据。

```typescript
type AppState = {
  // 消息相关
  messages: Message[];
  streamingMessage: string | null;

  // MCP 相关
  mcp: {
    clients: Record<string, MCPClient>;
    tools: Tool[];
    commands: Command[];
    resources: ServerResource[];
    pluginReconnectKey: number;
  };

  // 工具权限
  toolPermissionContext: ToolPermissionContext;

  // 任务相关
  tasks: Record<string, TaskState>;

  // 插件相关
  plugins: {
    enabled: string[];
    disabled: string[];
    commands: Command[];
    errors: Record<string, string>;
    needsRefresh: boolean;
  };

  // 通知
  notifications: {
    current: Notification | null;
    queue: Notification[];
  };

  // 推测执行
  speculation: SpeculationState;

  // 会话钩子
  sessionHooks: SessionHooksState;

  // 其他状态
  verbose: boolean;
  // ...
};
```

状态按照功能领域进行了清晰的划分，每个子状态都有明确的职责。

### 7.3.2 状态默认值

**文件**: `src/state/AppStateStore.ts` (状态定义部分)

每个状态字段都需要有合理的默认值，确保应用在初始状态下能够正常渲染。

```typescript
const DEFAULT_APP_STATE: AppState = {
  messages: [],
  streamingMessage: null,

  mcp: {
    clients: {},
    tools: [],
    commands: [],
    resources: [],
    pluginReconnectKey: 0,
  },

  toolPermissionContext: {
    mode: 'default',
    additionalWorkingDirectories: new Map(),
    alwaysAllowRules: {},
    alwaysDenyRules: {},
    alwaysAskRules: {},
    isBypassPermissionsModeAvailable: true,
  },

  // ... 其他默认值
};
```

### 7.3.3 状态更新模式

组件更新状态时，应该使用函数式更新模式，确保状态更新的原子性。

```typescript
// 错误方式：直接修改
const messages = useAppState(s => s.messages);
messages.push(newMessage);  // 这不会触发重新渲染

// 正确方式：函数式更新
const setAppState = useSetAppState();
setAppState(prev => ({
  ...prev,
  messages: [...prev.messages, newMessage]
}));
```

这种模式确保了每次更新都是基于最新状态的不可变操作。

## 7.4 状态更新批处理

### 7.4.1 MCP 状态批处理

**文件**: `src/services/mcp/useManageMCPConnections.ts` (第 207-291 行)

对于频繁更新的状态，如 MCP 连接状态，使用批处理来优化性能。

```typescript
const MCP_BATCH_FLUSH_MS = 16;  // 约一帧的时间
const pendingUpdatesRef = useRef<PendingUpdate[]>([]);

const flushPendingUpdates = useCallback(() => {
  const updates = pendingUpdatesRef.current;
  pendingUpdatesRef.current = [];

  setAppState(prevState => {
    let mcp = prevState.mcp;
    for (const update of updates) {
      mcp = applyMcpUpdate(mcp, update);
    }
    return { ...prevState, mcp };
  });
}, [setAppState]);
```

批处理将多个快速连续的更新合并为一次状态更新，减少了渲染次数。

### 7.4.2 防抖和节流

对于某些高频触发的事件，如搜索输入，使用防抖来减少状态更新次数。

```typescript
// 防抖搜索
const [query, setQuery] = useState('');
const debouncedQuery = useDebounce(query, 300);

// useDebounce 实现
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}
```

## 7.5 MCP 系统架构

### 7.5.1 MCP 概述

MCP（Model Context Protocol）是一种标准化协议，用于 AI 系统与外部工具和服务的通信。通过 MCP，Claude Code 可以连接到各种外部系统，扩展其能力范围。MCP 的核心概念包括：服务器（Server），提供特定功能的外部服务；客户端（Client），Claude Code 连接到服务器的实现；工具（Tool），服务器提供的可调用功能；资源（Resource），服务器提供的数据访问。

### 7.5.2 MCP 配置系统

**文件**: `src/services/mcp/types.ts` (第 9-169 行)

MCP 支持多种传输类型和配置方式。

```typescript
// 传输类型
type Transport = 'stdio' | 'sse' | 'sse-ide' | 'http' | 'ws' | 'sdk';

// 服务器配置类型
type McpStdioServerConfig = {
  type?: 'stdio',
  command: string;
  args: string[];
  env?: Record<string, string>;
};

type McpSSEServerConfig = {
  type: 'sse';
  url: string;
  headers?: Record<string, string>;
  oauth?: McpOAuthConfig;
};

type McpHTTPServerConfig = {
  type: 'http';
  url: string;
  headers?: Record<string, string>;
};

type McpWebSocketServerConfig = {
  type: 'ws';
  url: string;
  headers?: Record<string, string>;
};
```

不同的传输类型适用于不同的部署场景：stdio 用于本地进程；sse 和 http 用于远程 HTTP 服务；ws 用于 WebSocket 连接。

### 7.5.3 配置作用域

**文件**: `src/services/mcp/types.ts` (配置作用域)

MCP 配置支持多级作用域，允许在不同范围内定义服务器。

```typescript
type ConfigScope = 'local' | 'user' | 'project' | 'dynamic' | 'enterprise' | 'claudeai' | 'managed';

// 优先级（从高到低）
const CONFIG_SCOPE_PRIORITY = [
  'managed',    // 企业托管配置（最高优先级）
  'enterprise', // 企业级配置
  'claudeai',   // Claude.ai 连接器
  'project',    // 项目级 (.mcp.json)
  'user',       // 用户级（全局配置）
  'local',      // 本地配置
  'dynamic',    // 动态/插件配置（最低优先级）
];
```

当多个作用域中存在相同名称的配置时，优先级高的配置会覆盖优先级低的配置。

### 7.5.4 连接管理

**文件**: `src/services/mcp/useManageMCPConnections.ts` (第 143-1129 行)

useManageMCPConnections 是管理 MCP 服务器连接的 React Hook。

```typescript
// 连接状态
type MCPServerConnection =
  | ConnectedMCPServer    // 已连接
  | FailedMCPServer       // 连接失败
  | NeedsAuthMCPServer    // 需要认证
  | PendingMCPServer      // 连接中
  | DisabledMCPServer;   // 已禁用

// 主要功能
function useManageMCPConnections() {
  // 连接状态管理
  const [connections, setConnections] = useState<Record<string, MCPServerConnection>>({});

  // 连接到服务器
  const connect = async (config: McpServerConfig) => {...};

  // 断开连接
  const disconnect = async (serverName: string) => {...};

  // 重连
  const reconnect = async (serverName: string) => {...};
}
```

### 7.5.5 自动重连机制

**文件**: `src/services/mcp/useManageMCPConnections.ts` (第 354-461 行)

连接失败时，系统会使用指数退避策略自动重连。

```typescript
const MAX_RECONNECT_ATTEMPTS = 5;
const INITIAL_BACKOFF_MS = 1000;
const MAX_BACKOFF_MS = 30000;

const reconnectWithBackoff = async (serverName: string) => {
  for (let attempt = 1; attempt <= MAX_RECONNECT_ATTEMPTS; attempt++) {
    const backoffMs = Math.min(
      INITIAL_BACKOFF_MS * Math.pow(2, attempt - 1),
      MAX_BACKOFF_MS,
    );

    // 等待退避时间
    await new Promise(resolve => setTimeout(resolve, backoffMs));

    try {
      // 尝试重连
      await connect(serverName);
      return;  // 成功，退出重连循环
    } catch (error) {
      // 失败，继续下一次重连
    }
  }
};
```

指数退避策略平衡了快速重连和避免对服务器造成压力。

### 7.5.6 MCP 客户端

**文件**: `src/services/mcp/client.ts` (第 258-317 行)

MCP 客户端负责与服务器的实际通信。

```typescript
class MCPClient {
  private transport: Transport;
  private serverCapabilities: ServerCapabilities;

  // 初始化
  async initialize(): Promise<void> {
    // 发送初始化请求
    const response = await this.sendRequest('initialize', {
      protocolVersion: MCP_PROTOCOL_VERSION,
      capabilities: this.clientCapabilities,
      clientInfo: this.clientInfo,
    });
    this.serverCapabilities = response.capabilities;
  }

  // 调用工具
  async callTool(name: string, args: Record<string, unknown>): Promise<ToolResult> {
    return await this.sendRequest('tools/call', { name, arguments: args });
  }

  // 列出工具
  async listTools(): Promise<Tool[]> {
    const response = await this.sendRequest('tools/list');
    return response.tools;
  }

  // 读取资源
  async readResource(uri: string): Promise<ResourceContent> {
    return await this.sendRequest('resources/read', { uri });
  }
}
```

## 7.6 MCP 工具集成

### 7.6.1 MCPTool

**文件**: `src/tools/MCPTool/MCPTool.ts` (第 27-77 行)

MCPTool 是 Claude Code 中代表 MCP 工具的特殊工具类型。

```typescript
export const MCPTool = buildTool({
  isMcp: true,
  isOpenWorld() { return false; },
  name: 'mcp',
  maxResultSizeChars: 100_000,

  async checkPermissions(): Promise<PermissionResult> {
    return {
      behavior: 'passthrough',  // MCP 工具权限由 MCP 客户端管理
      message: 'MCPTool requires permission.',
    };
  },

  renderToolUseMessage,
  renderToolUseProgressMessage,
  renderToolResultMessage,
});
```

MCP 工具的特殊性在于它的工具定义是动态的，由连接的 MCP 服务器在运行时提供。

### 7.6.2 动态工具定义

**文件**: `src/services/mcp/client.ts` (MCP 工具定义)

当 MCP 服务器连接成功后，它的工具会被动态添加到 Claude Code 的工具列表中。

```typescript
// 从 MCP 服务器获取工具
const mcpTools = await mcpClient.listTools();

// 转换为 Claude Code 工具格式
const tools = mcpTools.map(mcpTool => buildTool({
  name: `mcp_${serverName}_${mcpTool.name}`,
  description: mcpTool.description,
  inputSchema: mcpTool.inputSchema,
  execute: async (input, context) => {
    return await mcpClient.callTool(mcpTool.name, input);
  },
}));
```

这种动态工具注册机制使得 Claude Code 可以透明地使用 MCP 服务器提供的功能。

## 7.7 权限系统详解

### 7.7.1 权��模式

**文件**: `src/types/permissions.ts` (第 16-38 行)

Claude Code 提供了多种权限模式，控制工具的执行行为。

```typescript
// 外部权限模式
const EXTERNAL_PERMISSION_MODES = [
  'acceptEdits',       // 自动接受编辑操作
  'bypassPermissions', // 完全绕过权限检查
  'default',          // 默认模式，每次请求确认
  'dontAsk',          // 拒绝所有请求
  'plan',             // 计划模式，只在计划阶段请求权限
] as const;

// 内部权限模式（包含实验性功能）
type InternalPermissionMode = ExternalPermissionMode | 'auto' | 'bubble';
```

不同的权限模式适用于不同的使用场景：默认模式适合学习环境；bypassPermissions 适合信任的 CI/CD 环境；plan 模式适合需要先查看计划再执行的工作流。

### 7.7.2 权限规则

**文件**: `src/types/permissions.ts` (第 75-79 行)

权限规则允许用户预先配置对特定工具的偏好。

```typescript
type PermissionRule = {
  source: PermissionRuleSource;     // 规则来源
  ruleBehavior: PermissionBehavior; // 'allow' | 'deny' | 'ask'
  ruleValue: PermissionRuleValue;   // { toolName, ruleContent? }
};

type PermissionRuleSource =
  | 'userSettings'    // 用户级设置
  | 'projectSettings' // 项目级设置（.claude/settings.json）
  | 'localSettings'   // 本地设置
  | 'flagSettings'    // 标志设置
  | 'policySettings'  // 企业策略
  | 'cliArg'          // 命令行参数
  | 'command'         // 会话命令
  | 'session';        // 会话级规则
```

权限规则的来源优先级从高到低是：命令行参数 > 会话命令 > 企业策略 > 项目设置 > 用户设置。

### 7.7.3 权限上下文

**文件**: `src/Tool.ts` (第 122-148 行)

权限上下文包含了做出权限决策所需的所有信息。

```typescript
type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode;  // 当前��限模式
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>;  // 额外工作目录
  alwaysAllowRules: ToolPermissionRulesBySource;  // 始终允许的规则
  alwaysDenyRules: ToolPermissionRulesBySource;   // 始终拒绝的规则
  alwaysAskRules: ToolPermissionRulesBySource;    // 始终询问的规则
  isBypassPermissionsModeAvailable: boolean;      // 是否可使用绕过模式
  isAutoModeAvailable?: boolean;                  // 自动模式是否可用
  prePlanMode?: PermissionMode;                   // 计划模式前的状态
}>;
```

权限上下文在程序启动时创建，并在整个会话期间保持不变。

### 7.7.4 权限决策流程

**文件**: `src/utils/permissions/permissions.ts` (第 400-500 行)

权限决策是一个多阶段的匹配过程。

```typescript
async function checkToolPermissions(
  tool: Tool,
  input: unknown,
  context: ToolUseContext,
): Promise<PermissionResult> {
  // 阶段 1：验证输入
  const validation = await tool.validateInput?.(input, context);
  if (validation?.result === false) {
    return { behavior: 'deny', message: validation.message };
  }

  // 阶段 2：工具特定的权限检查
  const toolPermission = await tool.checkPermissions(input, context);

  // 阶段 3：应用权限规则匹配
  const ruleResult = matchPermissionRules(tool, input, context.toolPermissionContext);

  // 阶段 4：根据模式和规则做出最终决策
  return resolvePermissionDecision(toolPermission, ruleResult, context);
}
```

每个阶段都可能改变最终的权限决策结果。

### 7.7.5 权限决策类型

**文件**: `src/types/permissions.ts` (第 171-266 行)

```typescript
type PermissionDecision =
  | PermissionAllowDecision   // 允许执行
  | PermissionAskDecision     // 请求用户确认
  | PermissionDenyDecision;  // 拒绝执行

type PermissionResult =
  | PermissionDecision
  | { behavior: 'passthrough', message: string };  // 传递给下一层处理
```

权限决策会被转换为相应的 UI 操作：允许执行会直接运行工具；请求确认会显示权限提示；拒绝执行会返回错误。

## 7.8 缓存与记忆化

### 7.8.1 MCP 认证缓存

**文件**: `src/services/mcp/client.ts` (第 258-317 行)

MCP 认证信息会被缓存以减少重复的磁盘读取。

```typescript
const MCP_AUTH_CACHE_TTL_MS = 15 * 60 * 1000;  // 15 分钟

function getMcpAuthCache(): Promise<McpAuthCacheData> {
  if (!authCachePromise) {
    authCachePromise = readFile(getMcpAuthCachePath(), 'utf-8')
      .then(data => jsonParse(data))
      .catch(() => ({}));
  }
  return authCachePromise;
}
```

### 7.8.2 状态记忆化

模块级别的状态会被记忆化，避免重复计算。

```typescript
// 模块级缓存
const commandsCache = new Map<string, Command[]>();

function getCommands(cwd: string): Command[] {
  if (commandsCache.has(cwd)) {
    return commandsCache.get(cwd)!;
  }

  const commands = loadCommands(cwd);
  commandsCache.set(cwd, commands);
  return commands;
}
```

## 7.9 并发控制

### 7.9.1 MCP 连接并发

**文件**: `src/services/mcp/useManageMCPConnections.ts`

使用 p-map 控制并发连接的 MCP 服务器数量。

```typescript
import pMap from 'p-map';

// 限制并发数为 5
await pMap(serverConfigs, async (config) => {
  await connectToServer(config);
}, { concurrency: 5 });
```

### 7.9.2 工具执行并发

工具系统也支持并行执行多个独立的工具调用。

```typescript
// 并行执行独立工具
const results = await Promise.all(
  toolBlocks.map(toolBlock => executeTool(toolBlock, context))
);
```

## 7.10 本章小结

本章我们详细分析了 Claude Code 的状态管理和 MCP 系统，包括：Zustand 风格 Store 的实现和 React 集成；应用状态的完整结构和默认值；状态更新的批处理和防抖机制；MCP 系统的配置、连接和工具管理；权限系统的多层设计和决策流程；缓存与记忆化优化；以及并发控制策略。

这些系统共同构成了 Claude Code 的数据层，为上层的业务逻辑提供了稳定、高效的基础设施。

---

## 下一步

现在您已经了解了状态管理和 MCP 系统，接下来建议：

1. **阅读 [第八章：工具实现示例](./08-工具实现示例.md)** - 深入了解具体工具的实现
2. **阅读 [第九章：使用指南](./09-使用指南.md)** - 学习如何使用 Claude Code
3. **阅读 [第十章：开发指南](./10-开发指南.md)** - 开始开发自己的扩展

---

> 💡 **提示**: 理解状态管理对于调试应用问题和扩展功能非常重要。