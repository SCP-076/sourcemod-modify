# SourceMod 修改指南

> 本文档记录了从官方 SourceMod 仓库 fork 后所做的所有自定义修改，包括修改原因、各文件作用和使用步骤。

---

## 一、修改概览

| # | 修改项 | 涉及文件 | 目的 |
|---|--------|---------|------|
| 1 | sourcepawn 子模块指向私有仓库 | `.gitmodules` | 使用自己修改版的 SP 编译器 |
| 2 | Release 编译改为手动触发 | `.github/workflows/build-release.yml` | 避免每次推送都自动发版 |
| 3 | Release 编译仅构建 L4D2 | `.github/workflows/build-release.yml` | 只需要求生之路2 |
| 4 | 脚本工具链编译改为手动触发 | `.github/workflows/scripting.yml` | 配合私有 SP 仓库，按需编译 |
| 5 | spcomp 容器镜像改为手动触发 | `.github/workflows/build-spcomp.yml` | 不需要自动推送容器镜像 |
| 6 | PR 检查仅构建 L4D2 | `.github/workflows/pr-checks.yml` | 减少 CI 耗时 |
| 7 | mock 测试改为手动触发 | `.github/workflows/mocktest.yml` | 仅 L4D2 不需要自动 mock 测试 |
| 8 | 添加私有仓库认证 | `build-release.yml`、`scripting.yml`、`pr-checks.yml`、`mocktest.yml` | 让 Actions 能拉取私有 SP 仓库 |

---

## 二、各文件详细说明

### 2.1 `.gitmodules` — 子模块配置

**修改内容：**

```ini
# 旧
[submodule "sourcepawn"]
    path = sourcepawn
    url = https://github.com/alliedmodders/sourcepawn
    shallow = true

# 新
[submodule "sourcepawn"]
    path = sourcepawn
    url = https://github.com/SCP-076/spp
    branch = spp-dev
    shallow = true
```

**原因：** 官方 sourcepawn 不包含自定义的 JIT/编译器修改，需要指向自己的私有 fork 仓库 `SCP-076/spp` 的 `spp-dev` 分支。

> **注意：** 如果在本地开发，需要手动更新子模块：
> ```bash
> git submodule sync
> git submodule update --init --recursive
> ```

---

### 2.2 `.github/workflows/build-release.yml` — 核心编译发布

**作用：** 编译完整的 SourceMod 发行包（编译器 + L4D2 扩展 + 核心），并自动创建 GitHub Release。

**修改内容：**

| 修改项 | 旧值 | 新值 |
|--------|------|------|
| 触发方式 | `push` 到 master/dev 分支自动触发 | `workflow_dispatch` 手动触发 |
| 手动参数 | 无 | 可选 `tag` 参数自定义版本号 |
| SDK 列表 | 动态扫描所有游戏（20+） | 硬编码仅 `l4d2` |
| 私有仓库认证 | 无 | checkout 前配置 git token |

**何时使用：**
- 修改了 sourcepawn 编译器 / JIT / 核心功能后
- 需要发布一个新的 SourceMod L4D2 安装包时

**如何使用（GitHub 网页操作）：**

1. 打开仓库页面 → 顶部 **Actions** 标签
2. 左侧选择 **Build Release**
3. 右侧点击 **Run workflow** 下拉按钮
4. `tag` 可选填（不填则自动使用 `product.version + git修订号`，如 `1.13.0.123`）
5. 点击绿色 **Run workflow** 按钮
6. 等待编译完成（约 10-20 分钟），自动在 Releases 页面生成安装包

---

### 2.3 `.github/workflows/scripting.yml` — 编译器快速验证

**作用：** 仅编译 SourcePawn 编译器工具链（spcomp），不包含游戏扩展。用于快速验证编译器能否通过构建。

**修改内容：**

| 修改项 | 旧值 | 新值 |
|--------|------|------|
| 触发方式 | `push` + `schedule` 定时 + `workflow_dispatch` | 仅 `workflow_dispatch` 手动触发 |
| 私有仓库认证 | 无 | checkout 前配置 git token |

**何时使用：**
- 修改 sourcepawn 编译器后，想快速验证编译是否通过（几分钟完成）
- 不想等完整 build-release（十几分钟）的时候

**如何使用：**
- 同 build-release，在 Actions → SourcePawn scripting → Run workflow

---

### 2.4 `.github/workflows/build-spcomp.yml` — Docker 容器镜像

**作用：** 将 spcomp 编译器打包成 Docker 容器镜像推送到 ghcr.io。

**修改内容：**

| 修改项 | 旧值 | 新值 |
|--------|------|------|
| 触发方式 | `push` + `pull_request` | 仅 `workflow_dispatch` 手动触发 |

**何时使用：**
- 一般**不需要**使用
- 只有想发布新的 spcomp Docker 镜像给别人用时才需要

---

### 2.5 `.github/workflows/pr-checks.yml` — Pull Request 编译检查

**作用：** 当有人提交 Pull Request 时，自动编译验证代码能否通过。

**修改内容：**

| 修改项 | 旧值 | 新值 |
|--------|------|------|
| SDK 列表 | `episode1,css,tf2,l4d2,csgo,dods` | 仅 `l4d2` |
| 私有仓库认证 | 无 | checkout 前配置 git token |

**触发方式未变：** 仍然是 PR 时自动触发，不需要手动操作。

---

### 2.6 `.github/workflows/mocktest.yml` — hl2sdk-mock 测试

**作用：** 使用 mock（模拟）SDK 进行不依赖真实游戏引擎的自动化测试。

**修改内容：**

| 修改项 | 旧值 | 新值 |
|--------|------|------|
| 触发方式 | `push` + `pull_request` | 仅 `workflow_dispatch` 手动触发 |
| 私有仓库认证 | 无 | checkout 前配置 git token |

**何时使用：**
- 一般**不需要**使用（仅 L4D2 不需要 mock 测试）
- 如果以后需要运行不依赖真实引擎的 SourceMod 单元测试时手动触发

---

## 三、GitHub Secrets 配置

为使 GitHub Actions 能拉取私有仓库 `SCP-076/spp`，需要在仓库中配置一个 Secret：

| Secret 名称 | 值 | 说明 |
|-------------|-----|------|
| `PRIVATE_SP_REPO_TOKEN` | `github_pat_xxx...` | 有 `SCP-076/spp` 仓库 Contents 读权限的 Personal Access Token |

**创建 Token 步骤：**

1. GitHub → 右上角头像 → **Settings** → **Developer settings** → **Personal access tokens** → **Fine-grained tokens**
2. **Resource owner**: 选择 `SCP-076`
3. **Repository access**: Only select repositories → 选择 `SCP-076/spp`
4. **Permissions** → Repository permissions → **Contents** → Read-only
5. 生成后复制 token

**添加到仓库：**

1. 打开本仓库 → **Settings** → **Secrets and variables** → **Actions**
2. **New repository secret**
3. Name: `PRIVATE_SP_REPO_TOKEN`
4. Secret: 粘贴 token

---

## 四、推荐操作流程

### 日常修改 sourcepawn 后的完整流程：

```
修改 SCP-076/spp 仓库 (spp-dev 分支)
         │
         ├─ (可选) 手动触发 scripting.yml
         │    └─ 目的：快速验证编译器能否编译通过（3-5 分钟）
         │
         └─ 手动触发 build-release.yml
              └─ 目的：生成完整 L4D2 SourceMod 安装包并发布 Release（10-20 分钟）
```

### 如果只修改了 SourceMod 核心（不涉及编译器）：

```
修改 sourcemod-modify 仓库 → 手动触发 build-release.yml
```

---

## 五、注意事项

1. **Token 过期：** Fine-grained PAT 有过期时间，过期后需要重新生成并更新 Secret
2. **SP 仓库改名：** 如果 `SCP-076/spp` 改名或换分支，需要同步修改 `.gitmodules` 中 `url` 和 `branch` 字段以及各 workflow 中 git config 的 `insteadOf` URL
3. **本地子模块同步：** 修改 `.gitmodules` 后，本地执行 `git submodule sync && git submodule update --init --recursive` 生效
4. **SDK 变更：** 如果以后需要支持更多游戏，修改 `build-release.yml` 中 `sdk_list` 的输出和 `pr-checks.yml` 中 `SDKS` 环境变量
5. **版本号：** `product.version` 文件控制基础版本号（当前 `1.13.0`），每次发版前可按需更新
