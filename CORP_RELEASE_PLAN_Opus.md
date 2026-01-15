# OpenCode 企业内网发布 + 仅 Copilot + 最小化裁剪方案

---

## 快速开始：最小改动方案（累进式组合）

> **核心原则**：按需组合，每项都是独立的增量改动

### 第 1 层：环境变量（零代码，必做）

```bash
# ~/.bashrc 或启动脚本
export OPENCODE_DISABLE_AUTOUPDATE=true      # 禁用版本检查
export OPENCODE_DISABLE_MODELS_FETCH=true    # 禁用模型列表刷新
export OPENCODE_DISABLE_LSP_DOWNLOAD=true    # 禁用 LSP 下载
export OPENCODE_DISABLE_DEFAULT_PLUGINS=true # 禁用默认插件安装
```

**效果**：禁用 4 个主要外联点，零代码改动

**注意**：禁用 LSP 下载后，如需 LSP 功能，需手动预置（见下方 LSP 处理方案）

---

### LSP 处理方案（禁用下载后）

LSP 服务器存储在 `~/.local/share/opencode/bin/`（XDG_DATA_HOME/opencode/bin）

#### 方案 A：完全不用 LSP（推荐）

```json
// opencode.json
{ "lsp": false }
```

代码补全、诊断等功能将不可用，但 AI 对话不受影响。

#### 方案 B：手动预置 LSP

```bash
# 1. 确定你需要的语言（示例：TypeScript、Go）
# 2. 在有网环境下载对应 LSP

# TypeScript (通过 npm)
mkdir -p ~/.local/share/opencode/bin/node_modules
cd ~/.local/share/opencode/bin
npm install typescript-language-server typescript

# Go (通过 go install)
GOBIN=~/.local/share/opencode/bin go install golang.org/x/tools/gopls@latest

# Clangd (下载二进制)
# https://github.com/clangd/clangd/releases
unzip clangd-linux-*.zip -d ~/.local/share/opencode/bin/

# 3. 打包整个 bin 目录用于离线分发
tar -czf opencode-lsp-bundle.tar.gz -C ~/.local/share/opencode bin/
```

#### 方案 C：系统 PATH 中的 LSP

代码会先检查 `Global.Path.bin`，如果不存在再检查系统 PATH。确保 LSP 在系统 PATH 中即可：

```bash
# 示例：gopls 已在系统中
which gopls  # /usr/local/bin/gopls -> 会被使用
```

#### LSP 存储路径速查

| LSP           | 存储位置                                                               |
| ------------- | ---------------------------------------------------------------------- |
| TypeScript    | `~/.local/share/opencode/bin/node_modules/typescript-language-server/` |
| ESLint        | `~/.local/share/opencode/bin/vscode-eslint/`                           |
| Pyright       | `~/.local/share/opencode/bin/node_modules/pyright/`                    |
| gopls         | `~/.local/share/opencode/bin/gopls`                                    |
| clangd        | `~/.local/share/opencode/bin/clangd` 或 `clangd_<version>/bin/clangd`  |
| rust-analyzer | 系统 PATH                                                              |
| jdtls (Java)  | `~/.local/share/opencode/bin/jdtls/`                                   |
| kotlin-ls     | `~/.local/share/opencode/bin/kotlin-ls/`                               |

---

### 第 2 层：配置文件（零代码，推荐）

```json
// opencode.json
{
  "autoupdate": false,
  "share": "disabled",
  "enabled_providers": ["github-copilot", "github-copilot-enterprise"]
}
```

**效果**：限制 provider 范围 + 禁用分享，与第 1 层叠加使用

---

### 第 3 层：代码补丁（按需选用）

| 改动             | 文件                    | 代码                                                | 效果                | 何时需要               |
| ---------------- | ----------------------- | --------------------------------------------------- | ------------------- | ---------------------- |
| 清空 BUILTIN     | `plugin/index.ts`       | `const BUILTIN: string[] = []`                      | 彻底禁用 npm 插件   | 环境变量不生效时       |
| 移除 CodexAuth   | `plugin/index.ts`       | 仅保留 `[CopilotAuthPlugin]`                        | 移除 Codex 认证入口 | 不需要 Codex           |
| 自定义 Client ID | `plugin/copilot.ts`     | `process.env.OPENCODE_COPILOT_CLIENT_ID \|\| "..."` | 支持企业 OAuth App  | 企业有自己的 OAuth App |
| 短路版本检查     | `installation/index.ts` | `return VERSION` 在函数开头                         | 彻底禁用升级检查    | 环境变量不够早         |

**补丁模板**（全部加起来 4 行）：

```diff
--- a/packages/opencode/src/plugin/index.ts
+++ b/packages/opencode/src/plugin/index.ts
-  const BUILTIN = ["opencode-anthropic-auth@0.0.9", "@gitlab/opencode-gitlab-auth@1.3.0"]
+  const BUILTIN: string[] = []
-  const INTERNAL_PLUGINS: PluginInstance[] = [CodexAuthPlugin, CopilotAuthPlugin]
+  const INTERNAL_PLUGINS: PluginInstance[] = [CopilotAuthPlugin]

--- a/packages/opencode/src/plugin/copilot.ts
+++ b/packages/opencode/src/plugin/copilot.ts
-const CLIENT_ID = "Ov23li8tweQw6odWQebz"
+const CLIENT_ID = process.env.OPENCODE_COPILOT_CLIENT_ID || "Ov23li8tweQw6odWQebz"

--- a/packages/opencode/src/installation/index.ts
+++ b/packages/opencode/src/installation/index.ts
 export async function latest(installMethod?: Method) {
+    return VERSION
```

---

### 推荐组合

| 场景                             | 推荐组合                            | 改动量   |
| -------------------------------- | ----------------------------------- | -------- |
| **个人使用，懒得管**             | 第 1 层                             | 0 行代码 |
| **个人使用，想干净点**           | 第 1 层 + 第 2 层                   | 0 行代码 |
| **企业内部，需要稳定**           | 第 1 层 + 第 2 层 + 第 3 层全部     | 4 行代码 |
| **企业内部，有自己的 OAuth App** | 上面 + CLIENT_ID 改动               | 4 行代码 |
| **需要通过安全审计**             | 以上全部 + Hard Delete（见第 6 节） | 大量     |

---

## 目录

0. [快速开始：推荐修改方案](#快速开始推荐修改方案按性价比排序)
1. [结论概述](#1-结论概述)
2. [Repo/Branch 模型](#2-repobranch-模型)
3. [Upstream 同步工作流](#3-upstream-同步工作流)
4. [内网发行工作流](#4-内网发行工作流)
5. [Copilot Business OAuth 研究结论](#5-copilot-business-oauth-研究结论)
6. [Provider 裁剪方案](#6-provider-裁剪方案)
7. [Proxy/证书方案](#7-proxy证书方案)
8. [外联代码识别与禁用](#8-外联代码识别与禁用)
9. [风险清单与回滚策略](#9-风险清单与回滚策略)
10. [实施里程碑](#10-实施里程碑)
11. [附录 A: 关键文件索引](#附录-a-关键文件索引)
12. [附录 B: 环境变量速查](#附录-b-环境变量速查)
13. [附录 C: 不确定点与验证方法](#附录-c-不确定点与验证方法)
14. [附录 D: 个人轻量级使用方案](#附录-d-个人轻量级使用方案零冲突维护)
15. [附录 E: 事实核验（源链接）](#附录-e-事实核验关键证据与源链接)
16. [附录 F: 最小化发行 MVP 清单](#附录-f-最小化发行-mvp-清单)
17. [附录 G: Copilot OAuth 最小改造点](#附录-g-copilot-oauth-最小改造点)

---

## 1. 结论概述

### 1.1 推荐的总体策略

**两层 Patch Stack + Tag-based Release**

| 层级                    | 策略                                    | 理由                                           |
| ----------------------- | --------------------------------------- | ---------------------------------------------- |
| upstream → my-local     | Patch Stack (git format-patch / git am) | 裁剪量大，需要可审计、可重放、可定位的变更记录 |
| my-local → corp-release | Merge-based 分支 + Cherry-pick          | 企业定制量小，更易于维护和回滚                 |

### 1.2 推荐的裁剪方式

**分阶段执行：先 Soft Delete，后 Hard Delete**

| 阶段    | 方式                               | 时间                            |
| ------- | ---------------------------------- | ------------------------------- |
| Phase 1 | Soft Delete（配置禁用 + 入口隐藏） | Week 1-2                        |
| Phase 2 | Hard Delete（真删代码）            | Week 3-4，待验证 Phase 1 稳定后 |

### 1.3 推荐的发布链路

```
upstream/dev → fetch → base/vX.Y.Z → patch stack → my-local/vX.Y.Z → merge → corp/vX.Y.Z → build → 内网发布
```

---

## 2. Repo/Branch 模型

```
                    [upstream/dev]  (原仓库 anomalyco/opencode dev 分支)
                          │
                          │ git fetch upstream
                          ▼
                    [base/vX.Y.Z]   (本地 tag，标记同步点)
                          │
                          │ git format-patch / git am (patch stack)
                          │ patches/ 目录存放补丁文件
                          ▼
                   [my-local/main]  (裁剪版本主分支)
                          │
                          │ git tag my-local/vX.Y.Z
                          ▼
                   [my-local/vX.Y.Z] (发布 tag)
                          │
                          │ git merge / cherry-pick
                          ▼
                    [corp/main]     (企业定制主分支)
                          │
                          │ git tag corp/vX.Y.Z
                          ▼
                    [corp/vX.Y.Z]   (企业内网发布 tag)
```

### 分支命名规范

| 分支/Tag | 命名                               | 说明                   |
| -------- | ---------------------------------- | ---------------------- |
| 上游跟踪 | `upstream/dev`                     | remote tracking branch |
| 同步基点 | `base/vX.Y.Z`                      | 标记从哪个上游版本开始 |
| 裁剪版本 | `my-local/main`, `my-local/vX.Y.Z` | 你的裁剪版本           |
| 企业版本 | `corp/main`, `corp/vX.Y.Z`         | 公司内网发布版本       |
| 补丁目录 | `patches/`                         | 存放所有 patch 文件    |

---

## 3. Upstream 同步工作流

### 3.1 初始化设置

```bash
# 添加上游仓库
git remote add upstream https://github.com/anomalyco/opencode.git

# 配置 rerere (记住冲突解决方案)
git config rerere.enabled true
git config rerere.autoupdate true

# 创建初始基点
git fetch upstream
git checkout -b my-local/main upstream/dev
git tag base/v1.1.21  # 当前版本
```

### 3.2 应用初始补丁栈

```bash
# 创建 patches 目录
mkdir -p patches/my-local

# 将你的所有修改生成为补丁（后续维护）
git format-patch base/v1.1.21..my-local/main -o patches/my-local/

# 补丁命名规范：
# 0001-disable-all-providers-except-copilot.patch
# 0002-remove-auto-upgrade-check.patch
# 0003-disable-telemetry.patch
# ...
```

### 3.3 同步上游更新

```bash
#!/bin/bash
# sync-upstream.sh

set -e

OLD_BASE=$(git describe --tags --match 'base/*' --abbrev=0)
NEW_VERSION="v1.2.0"  # 目标版本

# 1. 获取上游更新
git fetch upstream

# 2. 创建新基点
git tag base/${NEW_VERSION} upstream/dev

# 3. 备份当前分支
git branch -m my-local/main my-local/main-backup

# 4. 基于新基点创建新分支
git checkout -b my-local/main base/${NEW_VERSION}

# 5. 应用补丁栈
for patch in patches/my-local/*.patch; do
    echo "Applying: $patch"
    git am --3way "$patch" || {
        echo "Conflict in $patch - resolve and run: git am --continue"
        exit 1
    }
done

# 6. 验证补丁栈等价性
git range-diff ${OLD_BASE}..my-local/main-backup base/${NEW_VERSION}..my-local/main

# 7. 成功后清理
git branch -D my-local/main-backup

# 8. 打 tag
git tag my-local/${NEW_VERSION}
```

### 3.4 冲突处理原则

```bash
# 冲突解决后
git am --continue

# 如果补丁已被上游吸收
git am --skip
# 并记录到 patches/absorbed.md

# 如果补丁需要丢弃
mv patches/my-local/000X-xxx.patch patches/deprecated/
# 并记录原因到 patches/deprecated/README.md

# 重新生成补丁（解决冲突后）
git format-patch base/${NEW_VERSION}..my-local/main -o patches/my-local/
```

### 3.5 补丁栈管理文件

创建 `patches/MANIFEST.md`:

```markdown
# Patch Manifest

## Active Patches (my-local)

| #    | Patch                                | Purpose        | Status |
| ---- | ------------------------------------ | -------------- | ------ |
| 0001 | disable-all-providers-except-copilot | 仅保留 Copilot | Active |
| 0002 | remove-auto-upgrade-check            | 禁用自动升级   | Active |
| 0003 | disable-telemetry                    | 禁用遥测       | Active |

## Absorbed Patches (被上游吸收)

| Patch | Absorbed In | Date |
| ----- | ----------- | ---- |

## Deprecated Patches (已废弃)

| Patch | Reason | Date |
| ----- | ------ | ---- |
```

---

## 4. 内网发行工作流

### 4.1 构建流程

```bash
#!/bin/bash
# build-corp-release.sh

set -e

VERSION=$(cat packages/opencode/package.json | jq -r '.version')
BUILD_DIR="dist/corp-release"

# 1. 确保在 corp/main 分支
git checkout corp/main

# 2. 安装依赖（使用内网 registry）
export npm_config_registry="https://your-internal-registry.company.com"
bun install --frozen-lockfile

# 3. 构建
cd packages/opencode
bun run build

# 4. 打包
mkdir -p ${BUILD_DIR}
cp -r dist/* ${BUILD_DIR}/
cp package.json ${BUILD_DIR}/

# 5. 创建离线安装包
tar -czf opencode-corp-${VERSION}.tar.gz -C ${BUILD_DIR} .

# 6. 生成校验和
sha256sum opencode-corp-${VERSION}.tar.gz > opencode-corp-${VERSION}.tar.gz.sha256
```

### 4.2 预打包依赖（离线化关键）

```bash
#!/bin/bash
# prebundle-dependencies.sh

# 预先下载并打包所有运行时依赖，避免 runtime 动态下载

# 1. Provider SDK（关键：Copilot 使用内置 SDK，无需下载）
# 参考 packages/opencode/src/provider/provider.ts:44-67
# BUNDLED_PROVIDERS 已包含 @ai-sdk/github-copilot

# 2. 预安装默认插件（如果保留）
mkdir -p prebundled/plugins
cd prebundled/plugins

# 禁用默认插件或预下载
# 设置环境变量禁用: OPENCODE_DISABLE_DEFAULT_PLUGINS=true

# 3. LSP 服务器（如果保留 LSP 功能）
# 设置环境变量禁用下载: OPENCODE_DISABLE_LSP_DOWNLOAD=true
# 或预下载到 prebundled/lsp/

# 4. models.json（模型列表）
# 设置环境变量禁用: OPENCODE_DISABLE_MODELS_FETCH=true
# 预下载模型列表（仅 Copilot 相关）
curl -o prebundled/models.json "https://models.dev/api.json"
```

### 4.3 发布到内网

```bash
#!/bin/bash
# publish-internal.sh

VERSION=$1

# 方式 1: 内部 npm registry
npm publish --registry https://your-internal-registry.company.com

# 方式 2: 内部 artifact 仓库
curl -X PUT "https://artifactory.company.com/opencode/${VERSION}/opencode-corp-${VERSION}.tar.gz" \
    -T opencode-corp-${VERSION}.tar.gz

# 方式 3: 内部 Homebrew tap
# 更新 Formula 并推送到内部 git 仓库

# 方式 4: 容器镜像
docker build -t internal-registry.company.com/opencode:${VERSION} .
docker push internal-registry.company.com/opencode:${VERSION}
```

### 4.4 离线安装脚本

```bash
#!/bin/bash
# install-offline.sh
# 用户在内网机器上运行

PACKAGE_URL="https://artifactory.company.com/opencode/latest/opencode-corp.tar.gz"
INSTALL_DIR="${HOME}/.opencode/bin"

mkdir -p ${INSTALL_DIR}
curl -fsSL ${PACKAGE_URL} | tar -xz -C ${INSTALL_DIR}

# 添加到 PATH
echo 'export PATH="${HOME}/.opencode/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

echo "OpenCode installed successfully!"
```

---

## 5. Copilot Business OAuth 研究结论

### 5.1 支持情况

**结论：完全支持 Copilot Business**

OpenCode 的 Copilot 认证实现位于：`packages/opencode/src/plugin/copilot.ts`

```
认证流程：

[用户] → opencode auth connect github-copilot
         │
         ▼
[OpenCode] → POST https://github.com/login/device/code
              │
              │ client_id: Ov23li8tweQw6odWQebz
              │ scope: read:user
              ▼
[GitHub] ← 返回 device_code, user_code, verification_uri
              │
              ▼
[用户] → 打开 https://github.com/login/device 输入 user_code
              │
              ▼
[OpenCode] → 轮询 POST https://github.com/login/oauth/access_token
              │
              │ grant_type: urn:ietf:params:oauth:grant-type:device_code
              ▼
[GitHub] ← 返回 access_token (OAuth token)
              │
              ▼
[OpenCode] → AI 调用时使用 access_token 访问 https://api.githubcopilot.com
```

### 5.2 Copilot Business 兼容性

| 特性                           | 支持情况  | 说明                                      |
| ------------------------------ | --------- | ----------------------------------------- |
| Device Code Flow               | ✅ 支持   | 标准 OAuth 2.0 Device Authorization Grant |
| Copilot Business 授权          | ✅ 支持   | 只要用户有 Copilot 许可证                 |
| GitHub Enterprise Server       | ✅ 支持   | 可配置 `enterpriseUrl`                    |
| Enterprise Managed Users (EMU) | ⚠️ 需验证 | 可能受 EMU 策略限制                       |
| SAML SSO                       | ✅ 支持   | Device Code 流程支持 SSO                  |

### 5.3 GitHub Enterprise 配置

```typescript
// 认证时选择 "GitHub Enterprise" 选项
// packages/opencode/src/plugin/copilot.ts:110-143

// 配置选项
{
  "deploymentType": "enterprise",
  "enterpriseUrl": "company.ghe.com"  // 或 https://company.ghe.com
}
```

API 端点自动调整：

- Device Code: `https://{enterpriseUrl}/login/device/code`
- Access Token: `https://{enterpriseUrl}/login/oauth/access_token`
- Copilot API: `https://copilot-api.{enterpriseUrl}`

### 5.4 企业自定义 Client ID（高性价比改造）

企业内部 OAuth App 通常需要绑定特定的 Client ID。通过环境变量注入可实现**零代码修改**的扩展：

```bash
# 使用企业自定义 Client ID（可选，默认使用官方 ID）
export OPENCODE_COPILOT_CLIENT_ID="your-enterprise-client-id"
```

如需代码层面支持，仅需修改 `src/plugin/copilot.ts` 一行：

```typescript
// [MODIFIED] 读取环境变量，回退到默认 ID
const CLIENT_ID = process.env.OPENCODE_COPILOT_CLIENT_ID || "Ov23li8tweQw6odWQebz"
```

**改动性价比**：1 行代码 → 支持任意企业 OAuth App

### 5.5 潜在失败模式与排查

| 问题                 | 原因                    | 解决方案                                         |
| -------------------- | ----------------------- | ------------------------------------------------ |
| Device Code 请求失败 | Proxy 拦截 HTTPS        | 配置 `HTTPS_PROXY` 和 `NODE_EXTRA_CA_CERTS`      |
| 用户无法打开验证页   | 内网无法访问 github.com | 允许访问 github.com/login/device                 |
| Token 交换失败       | 网络超时                | 检查防火墙规则                                   |
| AI 调用 403          | Token 过期或无权限      | 重新认证：`opencode auth connect github-copilot` |
| EMU 用户无法认证     | 组织策略限制            | 联系 IT 管理员检查 OAuth 应用白名单              |

### 5.6 Token 存储位置

```bash
# Token 存储在
~/.local/share/opencode/auth.json

# 格式
{
  "github-copilot": {
    "type": "oauth",
    "refresh": "<access_token>",
    "access": "<access_token>",
    "expires": 0
  }
}
```

---

## 6. Provider 裁剪方案

### 6.1 现状分析

Provider 注册机制位于 `packages/opencode/src/provider/provider.ts`:

```typescript
// Line 44-67: BUNDLED_PROVIDERS (内置 SDK)
const BUNDLED_PROVIDERS = {
  "@ai-sdk/amazon-bedrock": createAmazonBedrock,
  "@ai-sdk/anthropic": createAnthropic,
  "@ai-sdk/azure": createAzure,
  "@ai-sdk/google": createGoogleGenerativeAI,
  "@ai-sdk/google-vertex": createVertex,
  // ... 等等
  "@ai-sdk/github-copilot": createGitHubCopilotOpenAICompatible,  // ← 保留这个
}

// Line 76-501: CUSTOM_LOADERS (自定义加载器)
const CUSTOM_LOADERS = {
  "github-copilot": async () => { ... },  // ← 保留这个
  "github-copilot-enterprise": async () => { ... },  // ← 保留这个
  "anthropic": async () => { ... },
  "openai": async () => { ... },
  // ... 等等
}
```

### 6.2 方案对比

#### 方案 A: Soft Delete（配置禁用 + 入口隐藏）

**改动点：**

1. **配置文件** (`opencode.json`)

```json
{
  "enabled_providers": ["github-copilot", "github-copilot-enterprise"]
}
```

2. **环境变量**

```bash
export OPENCODE_DISABLE_DEFAULT_PLUGINS=true
export OPENCODE_DISABLE_MODELS_FETCH=true
```

3. **UI 隐藏** - 可选，在 CLI 层过滤

**优点：**

- 上游同步冲突极小
- 可快速回滚（改配置即可）
- 保持代码完整性，便于调试

**缺点：**

- 攻击面未减少（代码仍存在）
- 打包体积较大
- 依赖仍会被安装

---

#### 方案 B: Hard Delete（真删代码）

**改动点：**

| 类别                  | 文件/目录                        | 操作                            |
| --------------------- | -------------------------------- | ------------------------------- |
| **Provider imports**  | `provider/provider.ts:17-39`     | 删除非 Copilot imports          |
| **BUNDLED_PROVIDERS** | `provider/provider.ts:44-67`     | 仅保留 `@ai-sdk/github-copilot` |
| **CUSTOM_LOADERS**    | `provider/provider.ts:76-501`    | 仅保留 `github-copilot*`        |
| **认证插件**          | `plugin/codex.ts`                | 删除                            |
| **默认插件**          | `plugin/index.ts:18`             | 清空 BUILTIN 数组               |
| **package.json 依赖** | `packages/opencode/package.json` | 移除非 Copilot SDK              |

**优点：**

- 攻击面最小
- 打包体积小
- 依赖更干净
- **合规优势**：保留代码即保留风险——审计人员只需检查 `package.json` 即可确认没有连接 AWS/Google 等云服务的能力

**缺点：**

- 上游同步冲突大
- 回滚复杂
- 需要维护大量补丁

**合规性论述**：

> 如果保留了 Google SDK，通过意想不到的配置泄露（如用户自定义 config），可能会触发连接。Hard Delete 确保"代码只要不在依赖里，就跑不起来"。

---

### 6.3 推荐路线图：先软后硬

```
Week 1-2: Soft Delete
    │
    ├─ 验证 Copilot 认证正常
    ├─ 验证 AI 调用正常
    ├─ 验证无外联
    │
Week 3-4: Hard Delete (可选)
    │
    ├─ 如果 Soft Delete 满足需求 → 保持现状
    └─ 如果需要更小体积 → 执行 Hard Delete
```

### 6.4 Soft Delete 实施补丁

创建 `patches/my-local/0001-soft-delete-providers.patch`:

```diff
--- a/packages/opencode/src/plugin/index.ts
+++ b/packages/opencode/src/plugin/index.ts
@@ -15,7 +15,8 @@ export namespace Plugin {
   const log = Log.create({ service: "plugin" })

-  const BUILTIN = ["opencode-anthropic-auth@0.0.9", "@gitlab/opencode-gitlab-auth@1.3.0"]
+  // CORP: Disable all builtin plugins except Copilot (which is internal)
+  const BUILTIN: string[] = []

   // Built-in plugins that are directly imported (not installed from npm)
-  const INTERNAL_PLUGINS: PluginInstance[] = [CodexAuthPlugin, CopilotAuthPlugin]
+  const INTERNAL_PLUGINS: PluginInstance[] = [CopilotAuthPlugin]
```

创建默认配置 `opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "enabled_providers": ["github-copilot", "github-copilot-enterprise"],
  "autoupdate": false,
  "share": "disabled"
}
```

---

## 7. Proxy/证书方案

### 7.1 现状支持矩阵

| 特性                  | 支持情况 | 实现位置         |
| --------------------- | -------- | ---------------- |
| `HTTPS_PROXY`         | ✅ 支持  | Bun 原生支持     |
| `HTTP_PROXY`          | ✅ 支持  | Bun 原生支持     |
| `NO_PROXY`            | ✅ 支持  | Bun 原生支持     |
| `NODE_EXTRA_CA_CERTS` | ✅ 支持  | Bun 原生支持     |
| localhost bypass      | ✅ 必需  | 官方文档明确说明 |

### 7.2 HTTP 客户端栈分析

OpenCode 使用 **Bun 原生 fetch**，继承标准 proxy 环境变量支持：

```typescript
// packages/opencode/src/plugin/copilot.ts
// 使用 fetch() - Bun 内置

// packages/opencode/src/provider/provider.ts:987-1007
// 自定义 fetch wrapper，仍基于原生 fetch

// packages/opencode/src/bun/index.ts
// bun add 命令也读取 proxy 环境变量 (Line 76-81)
```

### 7.3 企业 Proxy 配置

```bash
# ~/.bashrc 或 /etc/profile.d/opencode.sh

# Proxy 配置
export HTTPS_PROXY="http://proxy.company.com:8080"
export HTTP_PROXY="http://proxy.company.com:8080"

# 重要：localhost 必须 bypass（TUI 与本地 HTTP server 通信）
export NO_PROXY="localhost,127.0.0.1,::1"

# 自定义 CA 证书（SSL MITM 场景）
export NODE_EXTRA_CA_CERTS="/etc/pki/company-ca.pem"

# 可选：npm registry 配置（内网 registry）
export npm_config_registry="https://your-internal-registry.company.com"
```

### 7.4 验证 Checklist

```bash
#!/bin/bash
# verify-proxy.sh

echo "=== 环境变量检查 ==="
echo "HTTPS_PROXY: ${HTTPS_PROXY:-NOT SET}"
echo "HTTP_PROXY: ${HTTP_PROXY:-NOT SET}"
echo "NO_PROXY: ${NO_PROXY:-NOT SET}"
echo "NODE_EXTRA_CA_CERTS: ${NODE_EXTRA_CA_CERTS:-NOT SET}"

echo ""
echo "=== CA 证书检查 ==="
if [ -f "${NODE_EXTRA_CA_CERTS}" ]; then
    echo "CA cert exists: ${NODE_EXTRA_CA_CERTS}"
    openssl x509 -in ${NODE_EXTRA_CA_CERTS} -noout -subject -dates
else
    echo "WARNING: CA cert not found!"
fi

echo ""
echo "=== 本地服务器测试 ==="
curl -s -o /dev/null -w "%{http_code}" http://localhost:4096/health && echo " - localhost:4096 OK"

echo ""
echo "=== GitHub 连通性测试 ==="
curl -s -o /dev/null -w "%{http_code}" https://github.com/login/device && echo " - github.com OK"

echo ""
echo "=== Copilot API 测试 ==="
curl -s -o /dev/null -w "%{http_code}" https://api.githubcopilot.com && echo " - api.githubcopilot.com OK"
```

### 7.5 潜在问题与解决

| 问题             | 症状                              | 解决                                      |
| ---------------- | --------------------------------- | ----------------------------------------- |
| localhost 走代理 | TUI 卡死                          | 确保 `NO_PROXY=localhost,127.0.0.1`       |
| SSL 证书错误     | `UNABLE_TO_VERIFY_LEAF_SIGNATURE` | 设置 `NODE_EXTRA_CA_CERTS`                |
| 代理认证         | 407 错误                          | `HTTPS_PROXY=http://user:pass@proxy:8080` |

---

## 8. 外联代码识别与禁用

### 8.1 外联隐患详解（必须落地）

> **注意**：部分外联在配置关闭前已触发，需代码补丁提前 short-circuit

| 功能                | 隐患说明                                            | 代码位置                                                                                                                                                                                                                                                 | 禁用方式                                                            |
| ------------------- | --------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| **自动升级**        | `Installation.latest()` 在配置读取前已触发网络请求  | [upgrade.ts#L6-L23](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/cli/upgrade.ts#L6-L23), [installation/index.ts#L186-L239](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/installation/index.ts#L186-L239) | 需补丁提前 return                                                   |
| **models.dev 刷新** | 启动时/周期性 fetch 模型列表                        | [models.ts#L88-L108](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/provider/models.ts#L88-L108)                                                                                                                                   | `OPENCODE_DISABLE_MODELS_FETCH` 或改为仅 `/models --refresh` 时触发 |
| **插件下载**        | 内置插件启动时 `BunProc.install` 拉取 npm           | [plugin/index.ts#L18-L77](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/plugin/index.ts#L18-L77)                                                                                                                                  | 清空 BUILTIN 或 `OPENCODE_DISABLE_DEFAULT_PLUGINS`                  |
| **.opencode 依赖**  | `Config.installDependencies` 执行 `bun add/install` | [config.ts#L189-L210](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/config/config.ts#L189-L210)                                                                                                                                   | 需补丁改为显式开关                                                  |
| **LSP 下载**        | 多处直接 `fetch` GitHub releases                    | [server.ts#L1239-L1244](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/lsp/server.ts#L1239-L1244)                                                                                                                                  | `OPENCODE_DISABLE_LSP_DOWNLOAD` 或内网镜像                          |
| **.well-known**     | 远程配置拉取                                        | [config.ts#L41-L60](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/config/config.ts#L41-L60)                                                                                                                                       | 需新增 `OPENCODE_DISABLE_WELLKNOWN_FETCH` flag                      |

### 8.2 外联代码清单

| 功能                        | 代码位置                                                | 外联目标                      | 禁用方式                                                  |
| --------------------------- | ------------------------------------------------------- | ----------------------------- | --------------------------------------------------------- |
| **自动升级检查**            | `cli/upgrade.ts`, `installation/index.ts`               | npm/brew/github API           | `OPENCODE_DISABLE_AUTOUPDATE=true` 或 `autoupdate: false` |
| **模型列表刷新**            | `provider/models.ts:94`                                 | `https://models.dev/api.json` | `OPENCODE_DISABLE_MODELS_FETCH=true`                      |
| **Provider 动态安装**       | `bun/index.ts:63-129`, `provider/provider.ts:1023-1025` | npm registry                  | 使用内置 SDK + 内网 registry                              |
| **LSP 服务器下载**          | `lsp/server.ts` 多处                                    | GitHub releases               | `OPENCODE_DISABLE_LSP_DOWNLOAD=true`                      |
| **默认插件安装**            | `plugin/index.ts:47-76`                                 | npm registry                  | `OPENCODE_DISABLE_DEFAULT_PLUGINS=true`                   |
| **会话分享**                | `share/share.ts`, `share/share-next.ts`                 | `https://api.opencode.ai`     | `share: "disabled"`                                       |
| **Telemetry/OpenTelemetry** | `session/llm.ts:228`, `agent/agent.ts:270`              | 取决于配置                    | 默认禁用，除非显式配置                                    |
| **Web 搜索/代码搜索**       | `tool/websearch.ts`, `tool/codesearch.ts`               | `https://mcp.exa.ai`          | 需 API key，默认不启用                                    |

### 8.2 用户显式触发的外联（允许）

| 功能           | 触发条件                      | 外联目标                                            |
| -------------- | ----------------------------- | --------------------------------------------------- |
| Copilot 认证   | `/connect` 或 `opencode auth` | `github.com/login/device`, `github.com/login/oauth` |
| AI 调用        | 用户发送消息                  | `api.githubcopilot.com` (或企业端点)                |
| Web Fetch 工具 | 用户显式请求                  | 用户指定的 URL                                      |

### 8.3 禁用外联的配置模板

```json
{
  "$schema": "https://opencode.ai/config.json",
  "enabled_providers": ["github-copilot", "github-copilot-enterprise"],
  "disabled_providers": [],
  "autoupdate": false,
  "share": "disabled",
  "lsp": false,
  "experimental": {
    "openTelemetry": false
  }
}
```

```bash
# 环境变量（写入 /etc/profile.d/opencode.sh）
export OPENCODE_DISABLE_AUTOUPDATE=true
export OPENCODE_DISABLE_MODELS_FETCH=true
export OPENCODE_DISABLE_LSP_DOWNLOAD=true
export OPENCODE_DISABLE_DEFAULT_PLUGINS=true
```

### 8.4 Air-Gap 验证方案（不主动联网验证）

**测试环境**：Docker 容器，网卡配置为 `none` 或防火墙规则 `DROP ALL OUTBOUND`（仅放行 DNS 和特定 GHE IP）

```bash
#!/bin/bash
# air-gap-test.sh

echo "=== 1. 静默启动测试 ==="
# 预期：tcpdump 抓包显示 0 个外网 DNS 查询，0 个 HTTP 请求
timeout 10 tcpdump -i any -c 10 'port 80 or port 443' &
TCPDUMP_PID=$!
opencode --version
sleep 5
kill $TCPDUMP_PID 2>/dev/null
# 失败判定：如果尝试连接 registry.npmjs.org 或 api.github.com（版本检查），则测试失败

echo ""
echo "=== 2. 空闲挂机测试 ==="
# 启动 TUI 界面并保持 10 分钟，预期：无心跳包发送
echo "启动 opencode TUI，挂机 10 分钟后检查 tcpdump 输出..."

echo ""
echo "=== 3. 白名单连通性测试 ==="
# 配置 HTTPS_PROXY 后执行认证
export HTTPS_PROXY="http://proxy.company.com:8080"
opencode auth login
# 预期：流量仅流向 HTTPS_PROXY 指定的 IP
```

**验证标准**：

| 测试项           | 预期结果             | 失败判定                                               |
| ---------------- | -------------------- | ------------------------------------------------------ |
| 静默启动         | 0 个外网请求         | 任何到 `registry.npmjs.org` 或 `api.github.com` 的连接 |
| 空闲挂机 (10min) | 无心跳包             | 检测到周期性外发请求                                   |
| 白名单连通       | 流量仅走 HTTPS_PROXY | 直连外网 IP                                            |

### 8.5 禁用外联的补丁（Hard Delete 方式）

创建 `patches/my-local/0002-disable-network-calls.patch`:

```diff
--- a/packages/opencode/src/installation/index.ts
+++ b/packages/opencode/src/installation/index.ts
@@ -183,6 +183,9 @@ export namespace Installation {
   export const USER_AGENT = `opencode/${CHANNEL}/${VERSION}/${Flag.OPENCODE_CLIENT}`

   export async function latest(installMethod?: Method) {
+    // CORP: Disable version check
+    return VERSION
+
     const detectedMethod = installMethod || (await method())
     // ... rest of function
   }
 }
```

```diff
--- a/packages/opencode/src/provider/models.ts
+++ b/packages/opencode/src/provider/models.ts
@@ -86,6 +86,9 @@ export namespace ModelsDev {
   }

   export async function refresh() {
+    // CORP: Disable online model list refresh
+    return
+
     if (Flag.OPENCODE_DISABLE_MODELS_FETCH) return
     // ... rest of function
   }
 }
```

---

## 9. 风险清单与回滚策略

### 9.1 风险清单

| 风险                                      | 可能性 | 影响   | 缓解措施                                                   |
| ----------------------------------------- | ------ | ------ | ---------------------------------------------------------- |
| 上游同步冲突                              | 高     | 中     | 使用 Patch Stack + rerere，定期同步                        |
| **运行期动态下载触发外联**                | **高** | **高** | provider packages、插件、LSP、models.dev、自动升级均需禁用 |
| Runtime 动态下载失败                      | 高     | 高     | 使用内置 SDK + 禁用外部下载                                |
| Proxy SSL MITM 证书问题                   | 中     | 高     | 预先配置 `NODE_EXTRA_CA_CERTS`                             |
| Copilot Business 限制                     | 低     | 高     | 事先与 IT 确认 OAuth 应用白名单                            |
| EMU 用户无法认证                          | 低     | 高     | 检查组织策略，必要时申请白名单                             |
| **Copilot 设备码在企业 SSO/EMU 环境失败** | 中     | 高     | 使用 EMU 账号走 `/connect` 验证                            |
| 裁剪破坏核心功能                          | 中     | 高     | 分阶段裁剪，先软后硬                                       |
| 本地服务器被代理拦截                      | 中     | 高     | 确保 `NO_PROXY=localhost,127.0.0.1`                        |
| **OAuth scope 不足**                      | 低     | 中     | 当前仅 `read:user`，企业可能需要 `read:org`                |

### 9.2 回滚策略

```bash
# 回滚到上一个稳定版本
git checkout corp/vX.Y.Z-1

# 或回滚到 my-local 版本
git checkout my-local/vX.Y.Z

# 或回滚到纯上游版本
git checkout base/vX.Y.Z

# 重新构建并部署
./build-corp-release.sh
./publish-internal.sh
```

### 9.3 回滚点标记

每次发布前创建 tag:

```bash
git tag corp/vX.Y.Z-pre-release  # 发布前
git tag corp/vX.Y.Z              # 正式发布
```

---

## 10. 实施里程碑

### Week 1: 研究 + PoC

| 任务                       | 负责人 | 产出           |
| -------------------------- | ------ | -------------- |
| 克隆仓库，建立分支结构     | -      | Git 仓库       |
| 验证 Copilot Business 认证 | -      | 认证成功截图   |
| 配置 Proxy + CA 证书       | -      | 配置脚本       |
| Soft Delete PoC            | -      | 运行成功的 PoC |
| 文档内网环境配置要求       | -      | 配置文档       |

### Week 2: my-local 裁剪 + 离线安装

| 任务                  | 负责人 | 产出                          |
| --------------------- | ------ | ----------------------------- |
| 创建 Soft Delete 补丁 | -      | `patches/my-local/`           |
| 禁用所有外联          | -      | 验证脚本 + 结果               |
| 预打包依赖            | -      | `prebundled/` 目录            |
| 创建离线安装脚本      | -      | `install-offline.sh`          |
| 构建内网安装包        | -      | `opencode-corp-vX.Y.Z.tar.gz` |

### Week 3: corp-release 定制 + 权限/合规

| 任务         | 负责人 | 产出            |
| ------------ | ------ | --------------- |
| 企业配置模板 | -      | `opencode.json` |
| 安全审计     | -      | 安全审计报告    |
| 合规检查     | -      | 合规清单        |
| 集成测试     | -      | 测试报告        |
| 文档用户指南 | -      | 用户手册        |

### Week 4: 灰度发布 + 运维

| 任务                 | 负责人 | 产出      |
| -------------------- | ------ | --------- |
| 灰度发布（10% 用户） | -      | 监控数据  |
| 收集反馈             | -      | 反馈汇总  |
| 问题修复             | -      | Bug fixes |
| 全量发布             | -      | 公告      |
| 运维手册             | -      | 运维文档  |

---

## 附录 A: 关键文件索引

| 文件                                          | 说明                 |
| --------------------------------------------- | -------------------- |
| `packages/opencode/src/index.ts`              | CLI 主入口           |
| `packages/opencode/src/provider/provider.ts`  | Provider 注册中心    |
| `packages/opencode/src/plugin/copilot.ts`     | Copilot 认证插件     |
| `packages/opencode/src/plugin/index.ts`       | 插件加载器           |
| `packages/opencode/src/auth/index.ts`         | 认证存储             |
| `packages/opencode/src/installation/index.ts` | 安装/升级逻辑        |
| `packages/opencode/src/provider/models.ts`    | 模型列表（在线刷新） |
| `packages/opencode/src/bun/index.ts`          | Bun 包安装器         |
| `packages/opencode/src/flag/flag.ts`          | 功能开关             |
| `packages/opencode/src/config/config.ts`      | 配置加载             |
| `packages/opencode/src/mcp/index.ts`          | MCP 服务器管理       |
| `packages/opencode/src/share/`                | 会话分享功能         |

## 附录 B: 环境变量速查

| 变量                               | 作用                           | 建议值                |
| ---------------------------------- | ------------------------------ | --------------------- |
| `OPENCODE_DISABLE_AUTOUPDATE`      | 禁用自动升级                   | `true`                |
| `OPENCODE_DISABLE_MODELS_FETCH`    | 禁用模型列表刷新               | `true`                |
| `OPENCODE_DISABLE_LSP_DOWNLOAD`    | 禁用 LSP 下载                  | `true`                |
| `OPENCODE_DISABLE_DEFAULT_PLUGINS` | 禁用默认插件                   | `true`                |
| `OPENCODE_COPILOT_CLIENT_ID`       | 自定义 Copilot OAuth Client ID | 企业 App ID           |
| `HTTPS_PROXY`                      | HTTPS 代理                     | `http://proxy:8080`   |
| `HTTP_PROXY`                       | HTTP 代理                      | `http://proxy:8080`   |
| `NO_PROXY`                         | 代理白名单                     | `localhost,127.0.0.1` |
| `NODE_EXTRA_CA_CERTS`              | 自定义 CA 证书                 | `/path/to/ca.pem`     |

---

## 附录 C: 不确定点与验证方法

| 项目                  | 不确定点                    | 验证方法                               |
| --------------------- | --------------------------- | -------------------------------------- |
| EMU 用户认证          | EMU 策略可能阻止 OAuth 应用 | 使用 EMU 测试账户尝试认证              |
| 企业 Copilot API 端点 | GHE 部署可能有不同端点      | 查阅企业 GitHub 文档或咨询 GitHub 支持 |
| Bun proxy 行为        | 某些边缘情况可能绕过 proxy  | 使用 mitmproxy 抓包验证                |

---

## 附录 D: 个人轻量级使用方案（零冲突维护）

如果不追求公司发布，只是个人使用，以下方案可实现**零代码修改**或**最小改动**，同步时几乎无冲突：

### D.1 方案 A：纯配置（零冲突，推荐）

```bash
# ~/.bashrc 或 shell profile 中添加
export OPENCODE_DISABLE_AUTOUPDATE=true
export OPENCODE_DISABLE_MODELS_FETCH=true
export OPENCODE_DISABLE_LSP_DOWNLOAD=true
export OPENCODE_DISABLE_DEFAULT_PLUGINS=true
```

```json
// 项目根目录或 ~/.config/opencode/config.json
{
  "$schema": "https://opencode.ai/config.json",
  "autoupdate": false,
  "share": "disabled"
}
```

**优点**：上游随便 pull，永远不冲突

### D.2 方案 B：单文件最小补丁（1 个冲突点）

仅修改 `packages/opencode/src/plugin/index.ts`：

```diff
-  const BUILTIN = ["opencode-anthropic-auth@0.0.9", "@gitlab/opencode-gitlab-auth@1.3.0"]
-  const INTERNAL_PLUGINS: PluginInstance[] = [CodexAuthPlugin, CopilotAuthPlugin]
+  const BUILTIN: string[] = []
+  const INTERNAL_PLUGINS: PluginInstance[] = [CopilotAuthPlugin]
```

**同步策略**：

```bash
git stash && git pull origin dev && git stash pop
# 如果冲突，只有这一个文件，手动解决很快
```

### D.3 方案 C：Git Worktree 隔离

```bash
# 主目录保持纯净上游
cd ~/Workspace/opencode

# 创建独立工作目录
git worktree add ../opencode-mine dev

# 在 opencode-mine 里做小改动，主仓库保持干净
cd ../opencode-mine

# 同步时（零冲突）
cd ~/Workspace/opencode && git pull
cd ../opencode-mine && git merge dev
```

### D.4 方案选择指南

| 偏好                              | 推荐方案 |
| --------------------------------- | -------- |
| 完全不想处理冲突                  | 方案 A   |
| 想稍微干净点，能接受偶尔 1 个冲突 | 方案 B   |
| 想保持主仓库干净，分开管理        | 方案 C   |

---

## 附录 E: 事实核验（关键证据与源链接）

所有结论均有源代码/文档佐证：

### E.1 架构与设计

| 结论                                                | 证据来源                                                                                                                                    |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| OpenCode 明确定位 provider-agnostic，不绑定单一厂商 | [README.md#L101-L106](https://github.com/anomalyco/opencode/blob/dev/README.md#L101-L106)                                                   |
| Provider packages 动态安装并缓存到本地              | [troubleshooting.mdx#L120-L134](https://github.com/anomalyco/opencode/blob/dev/packages/web/src/content/docs/troubleshooting.mdx#L120-L134) |

### E.2 Copilot 认证

| 结论                                                | 证据来源                                                                                                                        |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| 设备码登录流程 `/connect → github.com/login/device` | [providers.mdx#L681-L705](https://github.com/anomalyco/opencode/blob/dev/packages/web/src/content/docs/providers.mdx#L681-L705) |
| Copilot OAuth 设备码实现                            | [copilot.ts#L11-L200](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/plugin/copilot.ts#L11-L200)          |
| Copilot enterprise 逻辑                             | [provider.ts#L693-L705](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/provider/provider.ts#L693-L705)    |
| enterpriseUrl 配置支持                              | [config.ts#L813-L834](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/config/config.ts#L813-L834)          |

### E.3 网络与代理

| 结论                                     | 证据来源                                                                                                                |
| ---------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| Proxy/Custom CA 环境变量与 NO_PROXY 要求 | [network.mdx#L10-L57](https://github.com/anomalyco/opencode/blob/dev/packages/web/src/content/docs/network.mdx#L10-L57) |
| 环境变量开关 (DISABLE\_\*)               | [cli.mdx#L550-L575](https://github.com/anomalyco/opencode/blob/dev/packages/web/src/content/docs/cli.mdx#L550-L575)     |

### E.4 外联代码位置（精确行号）

| 外联功能                 | 代码位置                              | 证据来源                                                                                                                                                                                                                                                 |
| ------------------------ | ------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 自动升级检查             | `upgrade.ts`, `installation/index.ts` | [upgrade.ts#L6-L23](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/cli/upgrade.ts#L6-L23), [installation/index.ts#L186-L239](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/installation/index.ts#L186-L239) |
| models.dev 刷新          | `provider/models.ts`                  | [models.ts#L88-L108](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/provider/models.ts#L88-L108)                                                                                                                                   |
| 插件 npm 下载            | `plugin/index.ts`                     | [plugin/index.ts#L18-L77](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/plugin/index.ts#L18-L77)                                                                                                                                  |
| .opencode 依赖安装       | `config/config.ts`                    | [config.ts#L189-L210](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/config/config.ts#L189-L210)                                                                                                                                   |
| LSP 下载                 | `lsp/server.ts`                       | [server.ts#L1239-L1244](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/lsp/server.ts#L1239-L1244)                                                                                                                                  |
| BunProc.install npm 拉取 | `bun/index.ts`                        | [bun/index.ts#L63-L99](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/bun/index.ts#L63-L99)                                                                                                                                        |
| .well-known 远程配置     | `config/config.ts`                    | [config.ts#L41-L60](https://github.com/anomalyco/opencode/blob/dev/packages/opencode/src/config/config.ts#L41-L60)                                                                                                                                       |

### E.5 已知问题

| 问题                                          | 证据来源                                                         |
| --------------------------------------------- | ---------------------------------------------------------------- |
| 企业环境中 opencode-copilot-auth 插件安装失败 | [Issue #2341](https://github.com/anomalyco/opencode/issues/2341) |

---

## 附录 F: 最小化发行 MVP 清单

### 保留项

- TUI/CLI 基础骨架 (`packages/opencode/src/cli`)
- 本地 HTTP server
- 会话/存储
- `/connect`, `/models` 命令
- Copilot OAuth 认证
- Copilot 请求链路
- `github-copilot-enterprise` provider

### 删除/禁用项

| 组件                | 禁用方式                           |
| ------------------- | ---------------------------------- |
| 其他 providers      | `enabled_providers` 配置           |
| 插件生态            | `OPENCODE_DISABLE_DEFAULT_PLUGINS` |
| MCP 远程连接        | 配置禁用                           |
| 分享/导入分享       | `share: "disabled"`                |
| 自动升级            | `OPENCODE_DISABLE_AUTOUPDATE`      |
| LSP 自动下载        | `OPENCODE_DISABLE_LSP_DOWNLOAD`    |
| OpenTelemetry       | 配置禁用                           |
| Models.dev 自动刷新 | `OPENCODE_DISABLE_MODELS_FETCH`    |

### 风险提示

| 风险                      | 说明                                             |
| ------------------------- | ------------------------------------------------ |
| Provider 列表为空         | TUI 可能不可用，需保留 Copilot                   |
| Models.dev 依赖切断       | 默认模型缺失，需预置静态模型列表                 |
| Copilot auth 依赖插件体系 | 不能删除 INTERNAL_PLUGINS 中的 CopilotAuthPlugin |

---

## 附录 G: Copilot OAuth 最小改造点

若当前实现不完全满足企业需求，以下是最小改动点：

| 改造项               | 改动位置                             | 改动量   | 说明                                       |
| -------------------- | ------------------------------------ | -------- | ------------------------------------------ |
| 扩展 OAuth scope     | `plugin/copilot.ts` device code 请求 | 1 行     | 增加 `read:org` 或企业要求的 scope         |
| 配置化 enterpriseUrl | `opencode.json` → provider options   | 0 行代码 | 避免交互式输入                             |
| Token 生命周期       | `plugin/copilot.ts`                  | 10-20 行 | 当前 `expires` 恒为 0 无 refresh，需添加   |
| 自定义 Client ID     | `plugin/copilot.ts`                  | 1 行     | 读取 `OPENCODE_COPILOT_CLIENT_ID` 环境变量 |

---

_文档版本: 1.2_
_生成日期: 2026-01-15_
_基于 OpenCode v1.1.21_
