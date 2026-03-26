# OpenClaw Fork 版本管理策略

> JiaoTou Studio 维护分支 — Shadow-Pirates/openclaw

---

## 一、分支与远程配置

```
Shadow-Pirates/openclaw (origin)     ← 推送到你的 GitHub fork
         ↑
         │ (fetch & merge)
         │
openclaw/openclaw (upstream)         ← 官方仓库，只读拉取
```

### 远程仓库

| 名称 | URL | 用途 |
|------|-----|------|
| `origin` | `https://github.com/Shadow-Pirates/openclaw.git` | 你的 fork，推送目标 |
| `upstream` | `https://github.com/openclaw/openclaw.git` | 官方仓库，来源 |

### 分支结构

```
main          ← 维护分支（跟踪 upstream/main + 自定义修改）
v2026.3.24    ← 官方 release tag（来自 upstream）
v2026.3.24-jt.1  ← Fork 版本 tag（包含 Chrome deduplication 修复）
```

---

## 二、目录结构要求

```
d:\jtstudio\              ← JiaoTou Studio 项目根目录
├── jt-studio\            ← 主应用（workspace:* 引用 openclaw）
│   ├── package.json      ← 依赖 "openclaw": "workspace:*"
│   └── scripts\
│       └── update-openclaw.js   ← 版本更新脚本
└── openclaw\             ← OpenClaw fork（必须是 jt-studio 的兄弟目录）
    ├── package.json      ← fork metadata + customFork 字段
    ├── FORK_STRATEGY.md  ← 本文档
    └── dist/             ← 构建产物（jt-studio 运行时引用）
```

**重要**：`openclaw` 目录必须与 `jt-studio` 同级目录，因为 `workspace:*` 引用的是相对路径。

---

## 三、自定义修改记录

| Commit | 描述 | 影响范围 | 状态 |
|--------|------|----------|------|
| `f9475e4c0` | fix(chrome): deduplicate concurrent launches | `src/browser/chrome.ts` `src/browser/server-context.availability.ts` | ✅ 已合并 |

**说明**：Chrome deduplication 修复通过在 `launchOpenClawChrome` 和 `ensureBrowserAvailable` 中添加 `pendingLaunches` / `pendingEnsures` Map，防止并发启动时出现 `PortInUseError`。这是纯追加式修改，不会与上游功能冲突。

---

## 四、package.json fork metadata

每次合并上游后，`package.json` 需要同步以下字段：

```json
{
  "name": "openclaw",
  "version": "<保留上游版本号>",        // 不改！
  "description": "[JiaoTou Studio Fork] Multi-channel AI gateway...",
  "homepage": "https://github.com/Shadow-Pirates/openclaw#readme",
  "bugs": {
    "url": "https://github.com/Shadow-Pirates/openclaw/issues"
  },
  "repository": {
    "type": "git",
    "url": "git+https://github.com/Shadow-Pirates/openclaw.git"
  },
  "customFork": {
    "maintainer": "JiaoTou Studio",
    "forkedFrom": "https://github.com/openclaw/openclaw",
    "customCommits": [
      {
        "hash": "f9475e4c0",
        "description": "fix(chrome): deduplicate concurrent launches"
      }
    ]
  }
}
```

**为什么 name 必须保持 `openclaw`**：
- jt-studio 用 `"openclaw": "workspace:*"` 引用
- pnpm workspace 要求包名与目录/引用名一致
- 改为 `@jt-studio/openclaw` 会破坏 `workspace:*` 解析

---

## 五、版本更新工作流

### 方式 1：自动脚本（推荐）

```bash
cd d:/jtstudio/jt-studio
node scripts/update-openclaw.js
```

脚本自动完成：
1. 拉取上游最新代码
2. 合并到 main 分支
3. 同步 fork metadata
4. 构建 openclaw (`pnpm build`)
5. 推送到 fork
6. 创建 fork 版本 tag
7. 重新安装 jt-studio 依赖

**其他用法**：
```bash
node scripts/update-openclaw.js --check   # 仅检查更新，不合并
node scripts/update-openclaw.js --merge   # 自动合并（可能有冲突）
```

### 方式 2：手动更新

```bash
# 1. 拉取上游最新代码
cd d:/jtstudio/openclaw
git fetch upstream

# 2. 切换到维护分支
git checkout main
git pull origin main   # 确保本地最新

# 3. 合并上游变更
git merge upstream/main
# 如果有冲突 → 参考"冲突处理"章节

# 4. 同步 fork metadata
# 编辑 package.json，替换 repository/homepage/bugs 为 fork 地址
# 保留上游的 version 和 name
git add package.json
git commit -m "chore: sync fork metadata"

# 5. 构建（关键！）
pnpm install
pnpm build

# 6. 推送到 fork
git push origin main

# 7. 创建 fork 版本 tag
git tag -a v<VERSION>-jt -m "JiaoTou Studio fork: v<VERSION>"
git push origin v<VERSION>-jt

# 8. 回到 jt-studio 重新安装
cd d:/jtstudio/jt-studio
pnpm install
```

### 方式 3：Rebase 模式（保持线性历史）

```bash
git fetch upstream
git rebase upstream/main
# 如果有冲突 → 参考"冲突处理"章节
git push --force-with-lease origin main
```

---

## 六、冲突处理

### 识别冲突

```bash
git status
# 冲突文件显示为：UU / AU / DU / AA / DD
```

### 常见冲突点及解决方案

| 文件 | 原因 | 解决方案 |
|------|------|----------|
| `src/browser/chrome.ts` | 上游可能修改了 Chrome 启动逻辑 | **优先保留** `pendingLaunches` Map + `doLaunchOpenClawChrome` 函数 |
| `src/browser/server-context.availability.ts` | 上游可能修改了 availability 逻辑 | **优先保留** `pendingEnsures` Map + `doEnsureBrowserAvailable` 函数 |
| `package.json` | 上游版本号变更 | **保留上游 version**，替换 `repository/homepage/bugs` |
| `src/gateway/*.ts` | API 签名变化 | 需要仔细审查，通常**保留上游** |
| `pnpm-lock.yaml` | 依赖变化 | **接受上游版本**，pnpm install 会自动更新 |
| `src/agents/**/*.ts` | Agent 核心逻辑 | **保留上游** |

### Chrome 相关冲突的详细处理

**场景 A：只有下游改了 `launchOpenClawChrome` 函数**
- 保留你的 `pendingLaunches` + `doLaunchOpenClawChrome`
- 检查函数签名是否变化，如果是新参数需要透传

**场景 B：上游也改了 Chrome 启动逻辑**
1. 先看上游改了什么（`git diff upstream/main -- src/browser/chrome.ts`）
2. 如果上游在函数内部做了修改：
   - 保留上游的函数体
   - 将你的 deduplication 逻辑包裹在外面：

```typescript
// ❌ 错误：直接替换整个函数
export async function launchOpenClawChrome(...) { /* 你的代码 */ }

// ✅ 正确：在原有逻辑外包裹 deduplication
const pendingLaunches = new Map<string, Promise<RunningChrome>>();
export async function launchOpenClawChrome(resolved, profile) {
  const cacheKey = `${profile.name}:${profile.cdpPort}`;
  const existing = pendingLaunches.get(cacheKey);
  if (existing) return await existing;
  const launch = doLaunchOpenClawChrome(resolved, profile);
  pendingLaunches.set(cacheKey, launch);
  try { return await launch; }
  finally { if (pendingLaunches.get(cacheKey) === launch) pendingLaunches.delete(cacheKey); }
}
```

### 解决冲突后

```bash
git add <解决的文件>
git add package.json   # 不要漏了 fork metadata
git commit -m "Merge upstream and resolve conflicts"
git push origin main
```

---

## 七、版本号规则

遵循上游版本号格式：`YYYY.M.D`

- **当前 fork 版本**：`2026.3.24` (对应 `v2026.3.24` upstream tag)
- **Fork 版本 tag**：`v2026.3.24-jt.1`（jt-studio fork 专用）
- **不维护独立版本号**：每次合并上游后，使用上游的版本号

---

## 八、验证清单

更新后验证以下内容：

```bash
# 1. openclaw 版本正确
cd d:/jtstudio/openclaw
node openclaw.mjs --version

# 2. 构建产物存在
ls dist/

# 3. jt-studio 依赖正常
cd d:/jtstudio/jt-studio
pnpm list openclaw   # 应该显示 workspace:*
```

---

## 九、注意事项

1. **不要直接在 `upstream/main` 上工作** — 所有修改都在本地 `main` 分支
2. **始终先 `fetch upstream`** — 确保有最新代码
3. **构建后才能使用** — `workspace:*` 引用需要 dist 产物
4. **使用 `--force-with-lease`** — 比 `--force` 更安全
5. **测试后再推送** — 确保 openclaw 可以正常构建和运行
6. **不要改包名** — `name` 必须是 `openclaw`，否则 `workspace:*` 引用失效
