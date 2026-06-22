# IvyeaOps Code Wiki

## 1. 项目概述

**IvyeaOps** 是一套开源、自托管的亚马逊运营工作台，将 Listing 制作、AI 生图、市场调研、深度分析、广告优化、AI 智能体、GBrain 知识库、Skill 工坊、服务器运维等运营全流程统一收进浏览器里。

### 1.1 技术栈

| 层次 | 技术 | 版本/说明 |
|------|------|-----------|
| 后端 | FastAPI | Python 3.9+ |
| 前端 | React + Vite | TypeScript |
| 数据库 | SQLite | 运行时数据存储 |
| 样式 | TailwindCSS | 3.4.x |
| 终端 | xterm.js | 浏览器内终端 |
| 代码编辑器 | CodeMirror | 6.x |

### 1.2 架构特点

- **本地部署**：数据与密钥留在自己服务器，不绑定第三方云
- **开箱即用**：预构建发行包已含前端 `client/dist`
- **可二次开发**：AGPL-3.0 开源，前后端按模块拆分
- **单点登录**：统一认证，侧边栏直达所有板块

---

## 2. 项目结构

```
IvyeaOps/
├── server/                # FastAPI 后端（Python）
│   ├── app/
│   │   ├── core/          # 配置、安全、集成
│   │   ├── routers/       # API 路由
│   │   ├── services/      # 业务逻辑服务
│   │   └── agents/        # 智能体后端
│   └── .env.example       # 环境变量示例
├── client/                # React + Vite 前端（TypeScript）
│   └── src/
│       ├── pages/workbench/   # 工作台板块页面
│       ├── agents/            # 智能体会话子应用
│       ├── components/        # 通用组件
│       ├── layouts/           # 布局组件
│       └── api/               # 类型化 API 客户端
├── amazon-image-workflow/ # 图片工作流模块（独立服务）
├── deploy/                # nginx / systemd / docker 部署模板
├── scripts/               # 安装/启动脚本
└── docs/                  # 文档
```

---

## 3. 后端架构

### 3.1 核心模块

#### 3.1.1 `server/app/core/` - 核心基础设施

| 文件 | 职责 |
|------|------|
| `config.py` | 应用配置加载（环境变量、`.env`） |
| `security.py` | 会话管理、认证依赖、权限控制 |
| `hashpw.py` | bcrypt 密码哈希 |
| `hub_settings.py` | 运行时配置管理 |
| `integrations.py` | 第三方服务集成 |
| `permissions.py` | 权限系统 |

#### 3.1.2 `server/app/routers/` - API 路由

| 路由文件 | 功能板块 | 权限 |
|----------|----------|------|
| `auth.py` | 用户认证 | 公开 |
| `market.py` | 市场调研 | 登录用户 |
| `listing.py` | Listing 工作台 | 模块权限 |
| `lingxing.py` | 领星 ERP | 管理员 |
| `brain.py` | GBrain 知识库 | 模块权限 |
| `skill.py` | Skill 中心 | 模块权限 |
| `agents/` | 智能体代理 | 模块权限 |
| `monitor.py` | 服务器监控 | 模块权限 |
| `terminal.py` | 终端管理 | 模块权限 |
| `deep_analysis.py` | 深度分析工具 | 模块权限 |
| `assistant.py` | AI 问答助手 | 登录用户 |
| `hub_settings.py` | 系统配置 | 管理员 |

#### 3.1.3 `server/app/services/` - 业务服务

| 服务文件 | 职责 |
|----------|------|
| `ai_synthesis_service.py` | AI 文本合成、多提供商降级链 |
| `sorftime_service.py` | Sorftime 数据源 API 封装 |
| `brain_chat_service.py` | GBrain 知识库对话服务 |
| `lingxing_service.py` | 领星 ERP 核心服务 |
| `lingxing_optimizer.py` | 领星广告优化引擎 |
| `skill_architect.py` | Skill 自动生成 |
| `agent_session_service.py` | 智能体会话管理 |
| `terminal_live_service.py` | 终端会话管理 |
| `token_archive.py` | Token 使用记录归档 |

### 3.2 关键类与函数

#### 3.2.1 安全模块 (`security.py`)

```python
def require_user(session: Optional[str]) -> str
```
- **功能**：验证会话，设置当前用户上下文，返回用户标识
- **参数**：`session` - 会话 Cookie 值
- **返回**：用户邮箱或管理员用户名
- **异常**：401（未认证）、403（已停用）

```python
def require_admin(user: str, session: Optional[str]) -> str
```
- **功能**：管理员专属依赖，验证用户为管理员
- **异常**：403（非管理员）

```python
def require_module(module_key: str) -> Callable
```
- **功能**：模块级权限控制工厂函数
- **参数**：`module_key` - 模块标识符（如 "tools", "agents"）
- **行为**：管理员无条件通过；普通用户需在权限列表中

#### 3.2.2 AI 合成服务 (`ai_synthesis_service.py`)

```python
async def synthesize(mode: str, query: str, marketplace: str, data: Dict) -> AsyncGenerator
```
- **功能**：多提供商降级链的市场调研报告生成
- **优先级**：DeepSeek → Apimart → Hermes CLI → Codex CLI → Claude CLI
- **安全特性**：非管理员用户强制使用 HTTP-only 提供商

```python
def _text_provider_chain() -> List[str]
```
- **功能**：获取配置的文本 AI 提供商链
- **安全控制**：非管理员用户自动过滤为 HTTP-only 提供商

#### 3.2.3 领星 ERP 服务 (`lingxing_service.py`)

```python
async def status() -> Dict[str, Any]
```
- **功能**：获取领星集成状态快照（不含敏感信息）

```python
async def openapi_verify(caller: str) -> Dict[str, Any]
```
- **功能**：验证领星 OpenAPI 凭证有效性

### 3.3 认证与权限体系

```
┌─────────────────────────────────────────────────────────────┐
│                      认证流程                               │
├─────────────────────────────────────────────────────────────┤
│  用户登录 → 验证密码 → 签发会话 Cookie → 后续请求携带 Cookie │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                      权限层级                               │
├─────────────────────────────────────────────────────────────┤
│  公开路由        → 无需认证（如 /api/auth/login）            │
│  登录用户        → require_user()（如市场调研）              │
│  模块权限        → require_module("xxx")（如智能体）         │
│  管理员         → require_admin()（如系统配置）              │
└─────────────────────────────────────────────────────────────┘
```

---

## 4. 前端架构

### 4.1 核心模块

#### 4.1.1 `client/src/pages/workbench/` - 工作台页面

| 页面组件 | 功能 |
|----------|------|
| `Home.tsx` | 运营驾驶舱（关键词监控、竞品监控、大盘流量） |
| `Market.tsx` | 市场调研（关键词/ASIN 分析） |
| `ListingWorkbench.tsx` | Listing 制作工作台 |
| `LingXing.tsx` | 领星 ERP 面板 |
| `Brain.tsx` | GBrain 知识库 |
| `SkillHub.tsx` | Skill 中心 |
| `Agents.tsx` | AI 智能体会话 |
| `ServerMonitor.tsx` | 服务器监控 |
| `DeepAnalysis.tsx` | 深度分析工具 |

#### 4.1.2 `client/src/agents/` - 智能体会话子应用

该模块是对 [claudecodeui](https://github.com/siteboon/claudecodeui) 的移植，提供完整的智能体交互体验：

| 子模块 | 职责 |
|--------|------|
| `chat/` | 聊天界面、消息处理、工具调用可视化 |
| `code-editor/` | 代码编辑器组件 |
| `file-tree/` | 文件树浏览器 |
| `git-panel/` | Git 版本控制面板 |
| `shell/` | 终端集成 |
| `task-master/` | 任务看板 |

#### 4.1.3 `client/src/api/` - API 客户端

| 文件 | 功能 |
|------|------|
| `client.ts` | axios 实例封装 |
| `projects.ts` | 项目管理 API |
| `agents.ts` | 智能体 API |
| `market.ts` | 市场调研 API |
| `settings.ts` | 系统配置 API |

### 4.2 关键组件

#### 4.2.1 Home 驾驶舱 (`Home.tsx`)

```tsx
function Home()
```
- **功能**：运营驾驶舱主页面
- **核心子组件**：
  - `KeywordMonitor` - 关键词监控
  - `AsinMonitor` - ASIN 监控（竞品/自有）
  - `CategoryWatch` - 类目大盘
  - `MarketTraffic` - 市场流量趋势
- **特性**：数据源切换、多站点支持（US/UK/DE/JP 等）

#### 4.2.2 市场调研 (`Market.tsx`)

- **功能**：关键词/ASIN 市场调研
- **核心能力**：
  - 实时 SSE 流式报告生成
  - Sorftime 数据预采集
  - AI 合成报告输出

#### 4.2.3 智能体会话 (`ChatInterface.tsx`)

- **功能**：原生智能体交互界面
- **特性**：
  - 流式输出渲染
  - 工具调用可视化
  - 会话 resume
  - 多提供商切换

### 4.3 状态管理

| Context/Store | 职责 |
|---------------|------|
| `AuthContext` | 用户认证状态 |
| `ThemeContext` | 主题配置 |
| `WebSocketContext` | WebSocket 连接管理 |
| `useSessionStore` | 会话状态 |

---

## 5. 智能体后端 (`server/app/agents/`)

### 5.1 架构概述

智能体后端提供 REST API 和 WebSocket 接口，替代原有的外部 Node 服务（:3002）。

### 5.2 路由结构

```
/api/agents/
├── /auth                  # 认证
├── /user                  # 用户信息
├── /projects              # 项目管理
├── /providers             # 提供商管理（sessions 嵌套）
├── /git                   # Git 操作
├── /taskmaster            # 任务看板
├── /settings              # 设置
├── /commands              # 命令执行
├── /mcp-utils             # MCP 工具
└── /projects/{id}/files   # 文件操作
```

### 5.3 关键服务

| 服务 | 职责 |
|------|------|
| `agent_registry.py` | 智能体发现与注册 |
| `agent_session_service.py` | 会话生命周期管理 |
| `ws.py` | WebSocket 处理（流式输出） |
| `shell_pty.py` | PTY 终端管理 |

---

## 6. 领星 ERP 集成

### 6.1 核心模块

| 文件 | 职责 |
|------|------|
| `lingxing_service.py` | 核心服务、凭证管理、审计日志 |
| `lingxing_openapi.py` | 领星 OpenAPI 封装 |
| `lingxing_optimizer.py` | 确定性规则引擎优化建议 |
| `lingxing_operate.py` | 受控写操作、三重复核 |
| `lingxing_dashboard.py` | 广告数据大盘聚合 |
| `lingxing_automation.py` | 周度自动化建议调度 |

### 6.2 安全架构

```
┌─────────────────────────────────────────────────────────────┐
│                   领星写操作安全护栏                         │
├─────────────────────────────────────────────────────────────┤
│  1. 双开关控制（总开关 + 写操作开关）                         │
│  2. 确定性护栏（白名单 / 幅度上限）                          │
│  3. 三重复核（三个独立 LLM 审查）                            │
│  4. 人工确认（最终确认执行）                                 │
│  5. 回滚快照（执行前备份）                                   │
│  6. 熔断机制（失败自动停止）                                 │
│  7. 全程审计（操作日志记录）                                 │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. 数据存储

### 7.1 SQLite 数据库

| 数据库文件 | 用途 |
|------------|------|
| `data/market_history.sqlite3` | 市场调研历史 |
| `data/brain_chat.sqlite3` | GBrain 对话记录 |
| `data/agent_sessions.sqlite3` | 智能体会话 |
| `data/users.sqlite3` | 用户管理（多用户模式） |
| `data/lingxing_audit.sqlite3` | 领星操作审计 |

### 7.2 运行时配置

- **`server/.env`**：启动配置（密钥、端口等）
- **`data/hub_settings.json`**：运行时配置（API 密钥、集成路径等）

---

## 8. 运行方式

### 8.1 开发环境

```bash
# 后端（开发模式）
cd server
pip install -r requirements.txt
IVYEA_OPS_DEV=1 python -m app.main

# 前端
cd client
npm install
npm run dev
```

### 8.2 生产部署

```bash
# Linux/macOS 一键安装（预构建包）
curl -L https://github.com/Hector-xue/IvyeaOps/releases/latest/download/IvyeaOps.zip -o IvyeaOps.zip
unzip IvyeaOps.zip && cd IvyeaOps
bash scripts/install.sh
bash scripts/start.sh
```

### 8.3 Docker 部署

```bash
docker compose up -d
```

### 8.4 访问地址

- 默认端口：`8001`
- 访问 URL：`http://127.0.0.1:8001`

---

## 9. 核心工作流示例

### 9.1 市场调研流程

```
用户请求 → 后端接收 → 判断 Hermes 是否可用
    ↓
Hermes 可用 → Hermes 原生调用（自带 MCP）→ 直接生成报告
    ↓
Hermes 不可用 → Sorftime 数据预采集 → AI 合成报告
    ↓
返回 SSE 流式结果 → 前端渲染
```

### 9.2 智能体会话流程

```
用户输入 → WebSocket 连接 → 智能体执行
    ↓
工具调用请求 → MCP 工具执行 → 返回结果
    ↓
流式输出 → 前端实时渲染
```

### 9.3 领星写操作流程

```
创建工单 → 三重复核（LLM）→ 人工确认 → 执行操作 → 审计记录
    ↓                      ↓
  拒绝                   成功/失败
    ↓                      ↓
  工单关闭              回滚/完成
```

---

## 10. 扩展与定制

### 10.1 添加新模块

1. 在 `server/app/routers/` 创建新路由文件
2. 在 `server/app/services/` 创建对应服务
3. 在 `server/app/main.py` 注册路由
4. 在前端 `client/src/pages/workbench/` 创建页面组件
5. 在侧边栏配置菜单项

### 10.2 配置 AI 提供商

通过 `data/hub_settings.json` 配置：

```json
{
  "text_ai_providers": "hermes,deepseek,assistant",
  "deepseek_api_key": "your_key",
  "assistant_provider": "deepseek",
  "assistant_model": "deepseek-chat",
  "assistant_api_key": "your_key"
}
```

---

## 11. 安全注意事项

1. **密钥保护**：`server/.env`、`data/hub_settings.json` 已 gitignore
2. **CSRF 防护**：Origin 白名单验证
3. **会话安全**：签名 Cookie，7 天过期
4. **权限隔离**：管理员与普通用户分离，模块级权限控制
5. **写操作保护**：领星 ERP 写操作需三重复核 + 人工确认
6. **数据隔离**：多用户模式下每个用户有独立数据目录

---

## 12. 许可证

**AGPL-3.0** (GNU Affero General Public License v3.0)

> 注：「智能体会话」板块移植自 AGPL-3.0 的 [claudecodeui](https://github.com/siteboon/claudecodeui)，按 copyleft 要求，整个 IvyeaOps 以 AGPL-3.0 发布。