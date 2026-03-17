# 🔗 外部连接审计报告

**审计时间：** 2026-03-17 13:17 GMT+8  
**审计人：** 艾普西隆 🛡️  
**协作人：** 贝塔 🔵  
**版本：** v1.0

---

## 一、审计范围

审计所有 OpenClaw 与外部服务的连接，包括 API 调用、认证方式、传输安全等。

---

## 二、外部连接清单

### 2.1 飞书 API（2个应用）

| 项目 | 详情 |
|------|------|
| **应用1** | Beta 飞书应用 |
| App ID | `cli_a939e52002b8dcb0` |
| 认证方式 | App Secret |
| 用途 | Beta 伙伴飞书通信 |
| 存储位置 | `~/.openclaw/.env` |
| 传输安全 | HTTPS (api.feishu.cn) |

| 项目 | 详情 |
|------|------|
| **应用2** | 主飞书应用 |
| App ID | `cli_a9386ea9bb389cc5` |
| 认证方式 | App Secret |
| 用途 | 主账号飞书通信 |
| 存储位置 | `~/.openclaw/workspace/.env` |
| 传输安全 | HTTPS (api.feishu.cn) |

**风险评估：** 🟡 中  
- 两个飞书应用分散在不同 .env 文件
- App Secret 明文存储（标准做法，但需确保文件权限）

### 2.2 Claude API

| 项目 | 详情 |
|------|------|
| 服务 | Anthropic Claude API |
| 认证方式 | API Key（通过 OpenClaw 内部配置） |
| 用途 | AI 模型调用 |
| 存储位置 | OpenClaw 内部管理 |
| 传输安全 | HTTPS (api.anthropic.com) |

**风险评估：** 🟢 低  
- OpenClaw 内部管理，未在 .env 中明文暴露

### 2.3 GitHub CLI

| 项目 | 详情 |
|------|------|
| 服务 | GitHub API |
| 认证方式 | ❌ **未认证** |
| 用途 | 代码管理、PR/Issue 操作 |
| 状态 | `gh auth status` 显示未登录 |

**风险评估：** 🔴 高（功能缺失）  
- GitHub CLI 未认证，无法执行代码管理任务
- 影响范围：约塔、伽马等需要 GitHub 操作的伙伴

### 2.4 Tavily API

| 项目 | 详情 |
|------|------|
| 服务 | Tavily 搜索 API |
| API Key | `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` |
| 认证方式 | API Key |
| 用途 | 网络搜索功能 |
| 存储位置 | `~/.openclaw/.env` |
| 传输安全 | HTTPS (api.tavily.com) |

**风险评估：** 🟡 中  
- API Key 明文存储（标准做法）
- 前缀 `tvly-dev` 表明是开发密钥，非生产密钥

### 2.5 BrowserWing

| 项目 | 详情 |
|------|------|
| 服务 | BrowserWing 浏览器自动化 |
| URL | `http://127.0.0.1:8080` |
| 认证方式 | 无（本地服务） |
| 用途 | 浏览器自动化 |
| 存储位置 | `~/.openclaw/.env` |

**风险评估：** 🟢 低  
- 本地服务，无外部暴露
- 使用 HTTP（本地可接受）

---

## 三、网络监听状态

```
协议  地址              端口    进程
TCP   127.0.0.1        18792   openclaw-gateway
TCP   127.0.0.1        18789   openclaw-gateway
TCP   127.0.0.1        18791   openclaw-gateway
TCP   127.0.0.53%lo    53      systemd-resolved
TCP   127.0.0.1        631     cupsd
```

**评估：** 🟢 良好  
- OpenClaw 网关仅监听 localhost（127.0.0.1）
- 无外部端口暴露
- DNS 和打印服务仅本地

---

## 四、安全发现

### 🔴 高风险

1. **GitHub CLI 未认证**
   - 影响：无法执行代码管理、PR 创建等任务
   - 建议：立即执行 `gh auth login`

### 🟡 中风险

2. **两个飞书应用分散配置**
   - 问题：凭证分散在两个 .env 文件
   - 建议：合并到单一 `~/.openclaw/.env`

3. **.env 文件未加入 .gitignore**
   - 问题：`.gitignore` 未包含 `.env` 文件
   - 建议：添加 `.env` 到 `.gitignore`

### 🟢 低风险

4. **所有外部连接均使用 HTTPS** ✅
5. **网关仅监听 localhost** ✅
6. **本地服务无外部暴露** ✅

---

## 五、连接架构图

```
OpenClaw Gateway (localhost:18789)
    ├──→ 飞书 API (api.feishu.cn) [HTTPS]
    ├──→ Claude API (api.anthropic.com) [HTTPS]
    ├──→ Tavily API (api.tavily.com) [HTTPS]
    ├──→ GitHub API (api.github.com) [❌ 未认证]
    └──→ BrowserWing (localhost:8080) [HTTP, 本地]
```

---

## 六、建议行动

| 优先级 | 行动项 | 负责人 | 截止时间 |
|--------|--------|--------|----------|
| 🔴 P0 | 认证 GitHub CLI | 艾普西隆 | 3/17 |
| 🟡 P1 | 合并 .env 文件 | 艾普西隆 | 3/18 |
| 🟡 P1 | 更新 .gitignore | 艾普西隆 | 3/18 |
| 🟢 P2 | 定期轮换 API Key | 梦想家 | 月度 |

---

*报告生成：2026-03-17 13:17 GMT+8*  
*下次审计：3/21（Week 1 末，M1 里程碑）*
